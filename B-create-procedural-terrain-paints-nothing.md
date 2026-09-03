---
id: B-create-procedural-terrain-paints-nothing
title: "landscape.create_procedural_terrain reports 'Layer painted successfully' when the layer does not exist on the material"
status: DONE
severity: High
category: bug
tags: [landscape, silent-false-success, layer, painting]
encounters: 1
lastSeen: 2026-08-12T00:00:00Z
---

# create_procedural_terrain reports success and paints nothing

Called with `layerName: "Grass"` on landscape `DotaTerrain`, the verb returned
`{"success": true, "message": "Layer painted successfully"}` and painted **nothing**.

Root cause: the landscape's material had **no `LandscapeLayerBlend` node and zero target
layers** — `target_layers` held only `__LANDSCAPE_VISIBILITY__`. There was no weightmap
layer named `Grass` to paint into, so the paint call resolved to a no-op that was
reported as success.

This is the same silent-elision class as the already-fixed `actor.spawn_batch` /
`foliage.add_instances` defects: an input the caller supplied is discarded and the
response claims the work was done.

Repro (2026-08-12, UE 5.8, `EAContentExamples58`): create a landscape whose material has
no `LandscapeLayerBlend`, then call `landscape.create_procedural_terrain` with any
`layerName`. Success is reported; the weightmap is untouched.

**Workaround:** rebuild the landscape material graph in place via
`MaterialEditingLibrary` from `python.execute` so the named layer actually exists, then
re-run.

**Fix:** before painting, resolve the named layer against the landscape's material target
layers. If it is absent, return a typed error (e.g. `LAYER_NOT_FOUND`) whose message
lists the layers that **are** available, instead of claiming success. Never report a
paint as successful when zero weightmap texels changed.

## Implementation (IN-REVIEW)

Real verb name: **`landscape.create_procedural_terrain`** (`Handlers/Environment/LandscapeHandler.cpp`).
`environment.*.create_procedural_terrain` cross-dispatches to it from `EnvironmentHandler.cpp`;
`environment.build.create_procedural_terrain` is a *different* verb (spawns a procedural mesh actor)
and was not touched.

Audited every path through the verb that could report a paint it did not perform. Each is now typed:

| Path | Before | After |
|---|---|---|
| `layerName` not a target layer on the material | auto-created a LayerInfo nothing samples, `success:true` | `LAYER_NOT_FOUND` + message listing available layers + error data `availableLayers[]` / `availableLayerCount` / `materialPath` |
| material declares no target layers at all (the reported repro) | same false success | `LANDSCAPE_MATERIAL_NO_LAYERS`, message naming `LandscapeLayerBlend` as the prerequisite |
| no landscape material assigned | same false success | `LANDSCAPE_NO_MATERIAL` (needed separately because `GetLandscapeMaterial()` silently substitutes the engine default surface material) |
| hollow landscape (no `ULandscapeComponent`s) | `GetLandscapeExtent` return value ignored → `MinX=MAX_int32`/`MaxX=MIN_int32` carried into region arithmetic (signed overflow feeding `TArray::Init`) | `LANDSCAPE_NO_COMPONENTS`, same diagnostic `landscape.edit` / `landscape.get_heights` emit |
| empty / inverted / out-of-bounds `region` | degenerate `SetAlphaData` rect, `success:true` | `INVALID_ARGUMENT` naming the actual extent |
| negative region coordinate | old `-1` sentinel silently replaced it with the full extent (full-landscape repaint) | `TOptional` + `LandscapeHeightStats::ResolveHeightRegion`, shared with the height verbs; a clamp that changed the request is reported in `warnings[]` |
| edit-layer landscape | unscoped `SetAlphaData` landed in no persistent edit layer, so the next weightmap regeneration composited the paint away | wrapped in `FScopedSetLandscapeEditingLayer(GetDefaultEditLayerGuid(...))` and `bUploadTextureChangesToGPU=true`, matching the height write in `landscape.edit` |
| `strength` out of 0..1, or quantizing to weight 0 | silently clamped / silently erased | reported in `warnings[]` (weight 0 is an ERASE, said so explicitly) |
| paint reported without evidence | no readback at all | `verify` (default true) settles the deferred regen and reads the weightmap back, reporting `sampledTexels` / `texelsWithWeight` / `texelsAtRequestedWeight`; a zero-weight readback raises a warning |

Also: the LayerInfo auto-create now fills the existing placeholder slot in `ULandscapeInfo::Layers`
instead of appending a duplicate entry with the same name, is preceded by `UpdateLayerInfoMap()`,
guards `SetAlphaData`'s `check(LayerInfo != nullptr)`, and warns that the created object is not a
shared `/Game` LayerInfo asset. The mutation follows the house dirty pattern via
`PinWright::MarkLevelActorModified(Landscape)` **before** the write (no `MarkRenderStateDirty()`:
`SetAlphaData`+`Flush` is an engine path that pushes its own render state, and
`EnvironmentDirtyUtils.h` reserves that call for raw field assignments).

Files:
- `Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp` — the verb + two helpers
  (`GetPaintableTargetLayerNames` reading `RetrieveTargetLayerNamesFromMaterials()` on 5.5+ /
  `GetLayersFromMaterial()` on 5.3–5.4, and `FormatLayerNameList`).
- `Source/PinWright/Private/Handlers/ErrorCodes.h` — `LAYER_NOT_FOUND`,
  `LANDSCAPE_MATERIAL_NO_LAYERS`, `LANDSCAPE_NO_MATERIAL`.
- `Source/PinWright/Private/Tests/Environment/TestLandscapePaintLayerHonesty.cpp` — 5 new tests.
- `Docs/wiki-src/landscape.md`, `Docs/error-code-catalog.md`.

**NOT compiled and NOT run** — the build and the live verification belong to the integration pass.

Runtime verification once the editor is up:
1. Negative (the original repro): `landscape.create_procedural_terrain {landscapeName:"DotaTerrain", layerName:"Grass"}`
   → must return `LANDSCAPE_MATERIAL_NO_LAYERS` (or `LAYER_NOT_FOUND` listing names, if that
   landscape's material has since been rebuilt), never `success:true`.
2. Positive: on a landscape whose material declares a layer, paint it and check
   `texelsWithWeight > 0`.
3. `Automation RunTests PinWright.landscape.create_procedural_terrain`.

## Related, found while fixing (separate tickets needed)

`material.authoring.configure_layer_blend` creates `UMaterialExpressionScalarParameter` nodes named
after the requested layers — **not** `LandscapeLayerBlend`/`LandscapeLayerWeight` entries. It
therefore does not create paintable target layers despite its name and its `add_landscape_layer`
cross-reference, so following the documented recovery path does not actually make a landscape
paintable.

**Now tracked as `B-configure-layer-blend-wrong-nodes` (OPEN, High).** It is the strong candidate
for the **upstream cause** of this ticket's repro: the caller used that verb, believed the landscape
was now paintable, and it never was — this ticket's false success was the second silent failure in
the chain, not the first. The detail above stays here as the record of where it was found.

The other half of that recovery path is also defective: `landscape.set_material` never rebuilds the
per-component landscape MICs (`LandscapeHandler.cpp:896-897` assigns then calls a bare
`PostEditChange()`, which misses the `MemberPropertyName == LandscapeMaterial` branch that calls
`UpdateAllComponentMaterialInstances()`), so even a correctly-built layer blend will not render
until that is fixed — `B-landscape-set-material-stale-mics` (OPEN, High).

## History
- `#1-initial-repro` `OPEN` reporter — "landscape.create_procedural_terrain returned success:true / 'Layer painted successfully' while painting nothing, because the landscape material had no LandscapeLayerBlend and no target layers besides __LANDSCAPE_VISIBILITY__. Must verify the named layer exists on the material and return a typed LAYER_NOT_FOUND listing available layers."
- `#2-typed-errors-and-layer-gate` `IN-REVIEW` developer — "Gated the paint on the layers the landscape MATERIAL declares (ALandscapeProxy::RetrieveTargetLayerNamesFromMaterials / GetLayersFromMaterial) instead of the stale ULandscapeInfo::Layers cache, and removed the auto-create-any-name branch. Added LAYER_NOT_FOUND (lists available layers in the message and in error data availableLayers[]), LANDSCAPE_MATERIAL_NO_LAYERS, LANDSCAPE_NO_MATERIAL, plus LANDSCAPE_NO_COMPONENTS and an empty-region INVALID_ARGUMENT for the other no-op paths. Also scoped SetAlphaData to the default edit layer (an unscoped weight write is composited away on an edit-layer landscape — a second silent-elision path), switched the edit interface to bUploadTextureChangesToGPU=true, added Modify() before the write, and added an opt-out weightmap readback (verify, default true) reporting texelsWithWeight. 5 automation tests in Tests/Environment/TestLandscapePaintLayerHonesty.cpp; landscape wiki page documents the LandscapeLayerBlend prerequisite and the new error table. NOT COMPILED, NOT RUN — needs the integration build plus the DotaTerrain negative-case check before a tester can move this to DONE."
- `#3-runtime-verified-and-committed` `DONE` tester — Built and runtime-verified.
  **The original repro no longer fake-succeeds:** `landscape.create_procedural_terrain
  {landscapeName:"DotaTerrain", layerName:"Grass"}` now returns `LANDSCAPE_MATERIAL_NO_LAYERS`
  naming the material `/Game/DotaBlockout/Materials/M_DotaTerrain`, with error data
  `availableLayers: []`, `availableLayerCount: 0` and the `LandscapeLayerBlend` prerequisite spelled
  out — never `{"success":true,"message":"Layer painted successfully"}` again.
  **`LAYER_NOT_FOUND` lists real names:** against a scratch landscape built with
  `/Game/ExampleContent/Landscapes/Materials/M_LandsacapeWeightDemo`, an unknown layer returns
  `LAYER_NOT_FOUND` with `availableLayers: ["Grass","Sand","Stone"]`, `availableLayerCount: 3`.
  **The prime suspect is CLEARED.** The `FScopedSetLandscapeEditingLayer` + `bUploadTextureChangesToGPU=true`
  weightmap change — flagged in the implementation notes as untested and the first thing to blame if
  the readback came back zero — works: painting `Grass` at `strength:1` returned `success:true` with
  `paintedTexels: 16129`, `sampledTexels: 16129`, **`texelsWithWeight: 16129`**,
  `texelsAtRequestedWeight: 16129`, `verified: true`. Not zero.
  **Other paths:** inverted region `{minX:100,maxX:1}` → `INVALID_ARGUMENT` "Empty paint region after
  clamping to landscape extent [0,0]..[126,126]"; `strength:0` succeeds but reports
  `texelsWithWeight: 0` plus an explicit warning that weight 0 ERASES the layer rather than painting
  it. A warning also fires that the auto-created LayerInfo lives in the actor's package rather than a
  shared `/Game` asset. All 5 new + 2 pre-existing
  `PinWright.landscape.create_procedural_terrain.*` automation tests pass. Scratch landscape and level
  deleted afterwards; project left with zero dirty packages. Committed as `6461f60d`.
  **Note on alias parity:** the implementation notes said to verify
  `environment.create_procedural_terrain`. That bare spelling is **not registered** and returns
  `UNKNOWN_ACTION` — it never existed. The real alias is the legacy `environment.build` dispatcher
  (`EnvironmentHandler.cpp:295`, sub `create_procedural_terrain` → `landscape.create_procedural_terrain`),
  and that dispatcher declares only `action` as a valid param, so it rejects `landscapeName`/`layerName`
  with `UNKNOWN_PARAMS` before ever cross-dispatching. That is pre-existing and untouched by this fix
  (`git diff` confirms no change to those lines); see the new spin-off note below.
