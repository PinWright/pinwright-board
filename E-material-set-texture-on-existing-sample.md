---
id: E-material-set-texture-on-existing-sample
title: "No material.authoring verb assigns a texture to an existing TextureSample node — forces property.set, and the SamplerType↔texture compatibility constraint is undiscoverable"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [material, material-authoring, add-texture-sample, texture-sample, sampler-type, property-set, fallback, docs]
---

# Assigning/changing a TextureSample's texture has no typed verb, and the SamplerType↔texture rule is undiscoverable

`material.authoring.add_texture_sample` only assigns a texture at **node-creation
time**, via its optional `texturePath` param
(`Material/MaterialAuthoringHandler.cpp` `add_texture_sample` handler, the
`texturePath`→factory-`Properties` read: it reads `texturePath` once, sets the
`Texture` property in the factory `Properties`, and creates the node — there is no
later write path). Until this ticket there was **no** `material.authoring.*` verb to
assign or change the texture on an **existing** TextureSample node (the new
`set_texture_sample_texture` added here is that verb). The sibling-named
`set_texture_parameter_value` is unrelated — it writes a *material
instance* texture **parameter** override, not the `Texture` property of a plain
`UMaterialExpressionTextureSample` on the parent graph.

So when a caller deliberately creates an *unassigned* sample first (a common,
legitimate intent — "leave it unset so I can point it at my texture later") and
then needs a texture on it (e.g. to compile, since UE rejects a textureless
TextureSample with `(Node TextureSample) Missing input texture`), the only path is
to drop off the typed surface onto a raw reflection write: `property.set` with the
engine UPROPERTY name `Texture` targeting the node. That works, but it (a) requires
knowing the native field name, (b) bypasses the `add_texture_sample` convention,
and (c) splits one coherent "make this sample point at texture X" intent across two
RPC families.

## Second axis: SamplerType↔texture compatibility is undiscoverable

The harder part of the same intent: once a caller is assigning a texture, the
TextureSample's `SamplerType` must **match the texture's derived sampler class** or
the material fails to compile. `add_texture_sample` lets you pick
`samplerType: LinearColor` (correct for AO/Roughness/Metallic linear data), and
`SamplerTypeValueFromName` (`Material/MaterialAuthoringHandler.cpp:240-251`) faithfully
stamps `SAMPLERTYPE_LinearColor` — but nothing on the surface tells the caller that
a `LinearColor` sampler then **requires a texture whose `SRGB=false`** (so the
engine's `GetSamplerTypeForTexture` derives `SAMPLERTYPE_LinearColor`). Assigning
the default sRGB engine texture produces the confusing compile error
`(Node TextureSample) Sampler type is Linear Color, should be Color for
/Engine/EngineResources/DefaultTexture.DefaultTexture` — which names the *node's*
sampler as the thing that's wrong, not the texture, so it points the caller in the
wrong direction.

There is no read surface for "what sampler class does texture X derive?" and no
docs note on the constraint, so the caller had to read the engine's
`GetSamplerTypeForTexture` mapping and `property.get`-probe **seven** engine
textures (`DefaultDiffuse_TC_Masks`, `Good64...Noise`, `GradientTexture0`,
`WhiteSquareTexture`, `DefaultBokeh`, `Black`, `FastBlueNoise_vec2`) on
`SRGB`/`Compression`/`LODGroup`/`VirtualTexture` before finding one
(`FastBlueNoise_vec2`: `TC_VectorDisplacementmap` + `SRGB=false`) that derives
`SAMPLERTYPE_LinearColor`.

## Why it's process friction (clean outcome, but a fallback + 7-probe hunt + 2 failed compiles)

The material was built correctly, but only after:

- compile #1 failed: `Missing input texture` (the unassigned sample the task asked
  for can't compile),
- `property.set Texture = DefaultTexture` (off-tool fallback, since no verb sets a
  texture on an existing node),
- compile #2 failed: `Sampler type is Linear Color, should be Color …` (the
  default texture is sRGB → derives `SAMPLERTYPE_Color`),
- an engine-source read of `GetSamplerTypeForTexture` + **6** `property.get` probes
  across 7 engine textures to find one that derives `SAMPLERTYPE_LinearColor`,
- `property.set Texture = FastBlueNoise_vec2`, then compile #3 clean.

Net cost: 1 off-tool fallback verb + 2 failed compiles + an engine-source detour +
6 probe calls, all to satisfy "put a linear texture on this sample."

Friction note verbatim: *"the task's 'leave texturePath unset' directly conflicts
with the clean-compile success check — a UE TextureSample with no texture fails
with 'Missing input texture', and assigning the sRGB DefaultTexture then failed the
SamplerType=LinearColor validation. I had to read the engine's
GetSamplerTypeForTexture mapping and property.get-probe several engine textures to
find one that derives SAMPLERTYPE_LinearColor (FastBlueNoise_vec2:
TC_VectorDisplacementmap + sRGB=false), and used property.set (not a dedicated
material-authoring verb) to assign the Texture on the existing node since
add_texture_sample only assigns at creation — both a discoverability/ergonomics gap
and a contradictory task spec."*

## What it should do

Pick downstream (docs is the cheap immediate win):

- **Ergonomic verb (the structural fix — IMPLEMENTED):** added
  `material.authoring.set_texture_sample_texture` — params `assetPath`, `nodeId`,
  `texturePath`, optional `samplerType` — that loads the texture, writes the node's
  `Texture` property through the same `LOAD_MATERIAL_OR_RETURN` (open-editor guard)
  plumbing the other authoring mutators use, and **auto-derives `SamplerType` from the
  texture** (via `UMaterialExpressionTextureBase::AutoSetSampleType` /
  `GetSamplerTypeForTexture`) when `samplerType` is omitted, so the sampler/texture pair
  is consistent by construction and the caller never touches `property.set` or guesses
  the field name. (Extending `add_texture_sample` to update-in-place by `nodeId` was the
  alternative shape; a distinct verb was chosen to keep the create vs. mutate intents
  separate.)
- **Docs (minimum, immediately):** on `docs/wiki-src/material.authoring.md`, add an
  `add_texture_sample` Limitations bullet: *`texturePath` is read only at node
  creation — there is no verb to change the texture on an existing TextureSample;
  use `property.set` with property `Texture` on the node id. A textureless
  TextureSample fails `compile_material` with `Missing input texture`. The node's
  `samplerType` must match the texture's derived sampler class: a `LinearColor`
  sampler requires a texture with `SRGB=false` (e.g. `TC_VectorDisplacementmap`),
  else compile fails with `Sampler type is Linear Color, should be Color`. Pick the
  texture's sRGB/compression to match the sampler (or vice-versa) before compile.*
- **Optional readback:** `get_material_node_details` already emits a TextureSample's
  *configured* `samplerType` + `texturePath` (shipped in
  `B-material-get-node-details-missing-pins-props`,
  `MGIR/MGIRExpressionUtils.h:262-268`) — but **not** the texture's *derived* sampler
  class, so the forward-lookup "what sampler does texture X require?" is still a probe
  sweep. The new verb's auto-derive (below) removes that hunt for the write path; a
  separate read field (e.g. `derivedSamplerType` on `get_material_node_details`, or a
  `texture.describe` field) would also serve callers who want to pick the sampler
  themselves — left as a lower-priority follow-up, not implemented here.

Filed E-/`docs` — the material is correctly built and compiles; the friction is
purely that (1) assigning a texture to an existing sample has no typed verb and
(2) the SamplerType↔texture constraint is undiscoverable until two failed compiles
and an engine-source read. Distinct from `E-texture-no-srgb-setter` (which is about
the *texture asset's* sRGB write helper) and from `F-image-brush-from-texture`
(widget brushes). Wiki page to improve: `docs/wiki-src/material.authoring.md`.

## History
- `#2-implemented` `IN-REVIEW` developer — Reworded the "Optional readback" axis to reflect that `B-material-get-node-details-missing-pins-props` already surfaces a TextureSample's *configured* `samplerType`/`texturePath` (only the texture-*derived* class is still missing); refreshed the stale `MaterialAuthoringHandler.cpp` line numbers (the file moved into the `Material/` subdir). Implemented the structural fix: added the `material.authoring.set_texture_sample_texture` verb (`Source/EditorAutomationRpcGateway/Private/Handlers/Material/MaterialAuthoringHandler.cpp`) — params `assetPath`, `nodeId`, `texturePath`, optional `samplerType`. It resolves the node via `FindExpressionByIdOrName`, rejects a non-`UMaterialExpressionTextureBase` node (`WRONG_NODE_TYPE`) and an unloadable texture (`TEXTURE_NOT_FOUND`), writes `TexExpr->Texture`, and when `samplerType` is omitted calls `AutoSetSampleType()` (engine `GetSamplerTypeForTexture`) to derive the sampler so the pair is consistent by construction — eliminating the property.set fallback + the 2 failed compiles + the 7-texture probe hunt. Goes through the open-editor-guarded `LOAD_MATERIAL_OR_RETURN` loader. Returns `{nodeId, texturePath, samplerType}`. Docs: added the `add_texture_sample` Limitations bullet + a `set_texture_sample_texture` H3 on `docs/wiki-src/material.authoring.md`. Regression test `Source/EditorAutomationRpcGateway/Private/Tests/Material/TestMaterialSetTextureSampleTexture.cpp` (3 cases: explicit-sampler assign-on-existing-node; auto-derive matches `GetSamplerTypeForTexture`; missing-node `NOT_FOUND`) — fails if the verb is reverted (handler-not-found + Texture never set).
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean material.authoring.add_component_mask task (focus material.authoring.add_component_mask, outcome clean) building /Game/Materials/M_PackedORM (Surface/DefaultLit/Opaque, packed ORM channel-split). `add_texture_sample` only assigns `texturePath` at creation (`MaterialAuthoringHandler.cpp:566-599`); no verb sets/changes the texture on an existing TextureSample node (`set_texture_parameter_value` at :1678 targets material *instance* parameters, not the node's `Texture` property). Task asked to leave the sample unassigned, which then can't compile (`Missing input texture`), and the default sRGB engine texture fails the `LinearColor` sampler check (`Sampler type is Linear Color, should be Color`). Caller fell back to `property.set Texture` on the node and `property.get`-probed 7 engine textures (after reading engine `GetSamplerTypeForTexture`) to find one deriving `SAMPLERTYPE_LinearColor` (`FastBlueNoise_vec2`: TC_VectorDisplacementmap + SRGB=false). Net: off-tool fallback verb + 2 failed compiles + engine-source detour + 6 probe calls. Source-confirmed: `add_texture_sample` reads `texturePath` once into the factory `Properties`; `SamplerTypeValueFromName` (:240-251) stamps the requested sampler but nothing validates/derives it against the texture. Distinct from `E-texture-no-srgb-setter` (texture-asset sRGB setter) and `F-image-brush-from-texture` (widget brushes). Fix: add `material.authoring.set_texture_sample_texture` (or update-in-place by nodeId) that auto-derives SamplerType from the texture; tag `docs` to add the `texturePath`-creation-only + SamplerType↔SRGB compatibility note on `docs/wiki-src/material.authoring.md`. Medium — recoverable, but cost 2 wasted compiles + a 7-texture probe sweep + an engine-source read on an otherwise clean build.
