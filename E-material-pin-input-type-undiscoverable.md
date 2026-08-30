---
id: E-material-pin-input-type-undiscoverable
title: "Material pin input type/dimension is undiscoverable — float2→float3 mismatch only surfaces as a cryptic compile error, forcing engine-source reading"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [material, material-authoring, connect-nodes, get-node-details, pin-type, noise, docs]
encounters: 2
lastSeen: 2026-07-01T00:17:53.1197644+03:00
---

# No way to learn a material input pin's expected type/dimension before compile

When wiring a material graph, there is no surface that tells the caller what
**type/dimension** an input pin expects. The pin's *name* is discoverable (the
wiki "Common target pins" table lists names; `get_material_node_details` echoes
the input names and their current connections after the
`B-material-get-node-details-missing-pins-props` fix), but the *expected type*
of each input — e.g. that `MaterialExpressionNoise`'s `Position` input requires
a **float3** — is reported nowhere:

- The `material.authoring.md` wiki "Common target pins" table (L20-36) lists
  only pin names per node, with no type column, and **does not list the Noise
  node at all**. Custom-HLSL *output* types are documented (L41:
  `Float1|Float2|Float3|Float4`) but no *input* pin types are.
- `material.authoring.connect_nodes` **accepts a dimension-mismatched wire
  silently** — wiring a float2 `TextureCoordinate` into Noise's float3
  `Position`/`World Position` input returns ok with no warning (this is the
  documented deferred-type-check behavior — compile errors surface on
  `compile_material`, not `connect_nodes`, per wiki L67 — so the *tool* is
  behaving correctly; the gap is purely discoverability).
- `material.authoring.get_material_node_details` reports the input *names* and
  *connections* but **not each pin's expected type/dimension**, so inspecting
  the Noise node (which this task did) still can't reveal the float3
  requirement.

The mismatch only surfaces at `compile_material` as
`no matching function for call to 'MaterialExpressionNoise'` — an HLSL-overload
error that names the function, not the offending pin or the expected dimension,
so it doesn't point the caller at the fix (promote the float2 to float3 via an
`AppendVector` + a `Constant` for the Z component).

## Why it's process friction (clean outcome, but a compile-fail + engine-source detour)

The task finished cleanly, but only after a wasted authoring round-trip:

- The whole Noise pipeline was wired (`TexCoord → Noise Position`,
  `Noise → Multiply A`, `RoughnessScale → Multiply B`,
  `Multiply → Roughness`, `BaseColorTint → BaseColor`), and the caller even
  ran two `get_material_node_details` inspections on the Noise and Multiply
  pins beforehand — none of which surfaced the type requirement.
- First `compile_material` failed with the cryptic overload error.
- The caller had to **read the engine's `MaterialExpressionNoise.cpp` /
  `Common.ush`** to learn the `Position` input needs a 3-vector.
- Then insert an `add_material_node` Constant (float1 Z) + `add_append`
  AppendVector to promote the UVs to float3, re-wire (3 more `connect_nodes`),
  and recompile clean.

Friction note verbatim: *"Struggled once: wiring the float2 TextureCoordinate
directly into the Noise 'World Position' input compiled to invalid HLSL ('no
matching function for call to MaterialExpressionNoise' — overloads require
float3); had to read the engine's MaterialExpressionNoise.cpp/Common.ush to
learn the Position input needs a 3-vector, then insert an AppendVector +
Constant to promote the UVs to float3 before recompiling clean."*

Net cost of the gap: 1 failed `compile_material` + an off-tool engine-source
read + 3 corrective node/connect calls, for a wiring that an up-front
"Position expects float3, promote a float2 with AppendVector" note would have
made first-try.

This is distinct from the judge-filed `E-material-main-output-no-node-readback`
(which is about reading back *Main-node output connections* and the
*shadingModel* readback asymmetry — the **verify/readback** stage). This ticket
is about the **author/wire stage**: the *input* pin type being undiscoverable
before compile.

## What it should do

Pick downstream (docs is the cheap immediate win):

- **Docs (minimum, immediately):** in `docs/wiki-src/material.authoring.md`,
  (a) add the **Noise** node to the "Common target pins" table with its
  `Position` (float3) / `FilterWidth` inputs, and (b) add a Limitations bullet:
  *Several expression inputs require a specific vector width — notably Noise's
  `Position` is a float3. `connect_nodes` does not type-check; a float2→float3
  mismatch only fails at `compile_material` with a
  `no matching function for call to 'MaterialExpression…'` overload error.
  Promote a float2 (e.g. a `TextureCoordinate`) to float3 with an
  `AppendVector` (A=the float2, B=a `Constant` float1 for Z) before wiring.*
  This is the single highest-value edit — it converts the engine-source detour
  into a one-line lookup.
- **Readback enrichment (optional):** add an expected-type field per input to
  `get_material_node_details`'s `inputs[]` (the `B-material-...-missing-pins-props`
  fix already walks `FExpressionInputIterator`; `Expr->GetInputType(Index)`
  / `MCT_Float3` etc. is available alongside `GetInputName`), so an inspection
  before wiring reveals the requirement without reading docs or source.
- **Or** make the compile error point at the pin: when a
  `no matching function for call to 'MaterialExpression…'` overload error is
  detected, append the offending expression's name + expected input
  dimensions to `compileErrors` so the message is self-diagnosing.

Filed E-/`docs` — the material is correctly wired and compiles; the friction is
purely that the input pin type is undiscoverable until a failed compile, and the
compile error doesn't name the fix. Wiki page to improve:
`docs/wiki-src/material.authoring.md`.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean material.authoring.add_noise task (focus material.authoring.add_noise, outcome ergo) authoring /Game/Materials/M_ProceduralConcrete (Surface/Opaque/DefaultLit). The Noise pipeline wired and inspected fine (2 get_material_node_details on Noise/Multiply pins), but the first compile_material failed with `no matching function for call to 'MaterialExpressionNoise'` because a float2 TextureCoordinate fed Noise's float3 Position input; connect_nodes accepted the mismatch silently and the compile error names the function, not the pin or expected dimension. The caller had to read engine MaterialExpressionNoise.cpp/Common.ush to learn Position needs float3, then add a Constant + AppendVector to promote to float3 and recompile clean (1 failed compile + off-tool source read + 3 corrective calls). The connect-accepts / compile-rejects behavior is correct (deferred type check, documented at wiki L67) — the gap is discoverability: input pin type/dimension is in neither the wiki pin table (Noise isn't listed; table has no type column) nor get_material_node_details. Distinct from judge-filed E-material-main-output-no-node-readback (Main-output readback + shadingModel asymmetry — the verify stage); this is the author/wire stage. Propose docs minimum (add Noise to the pin table + a "Position is float3; promote float2 via AppendVector; connect_nodes doesn't type-check" Limitations bullet on docs/wiki-src/material.authoring.md), optionally add per-input expected-type to get_material_node_details or make the overload compile error name the offending pin. Medium — recoverable, but cost a wasted compile + an engine-source detour on an otherwise first-try wiring.
- `#2-fresnel-pin-name-also-missing` `OPEN` reporter — Same root gap, a second axis: not just input *type* — the input pin **name** is also undiscoverable for convenience nodes the "Common target pins" table omits. Clean material.authoring.add_fresnel task (focus material.authoring.add_fresnel, outcome clean) authoring /Game/Materials/M_RimGlow (Surface/Translucent/DefaultLit rim-glow): RimPower scalar → Fresnel `ExponentIn`, Fresnel→Multiply→EmissiveColor, Fresnel→Opacity, 16 calls all first-try. But the table (`docs/wiki-src/material.authoring.md` L20-36) does **not list the Fresnel node at all**, so the `ExponentIn` input-pin name was unknown — the caller had to read the engine's `MaterialExpressionFresnel.h` to learn it before the `connect_nodes("RimPower -> Fresnel.ExponentIn")` could be written. Friction note verbatim: *"one quick read of the engine's MaterialExpressionFresnel.h confirmed the ExponentIn pin name so RimPower could drive the exponent naturally."* This corroborates the proposed docs fix (extend the "Common target pins" table) but broadens its scope beyond Noise: the table needs every `material.authoring.add_*` convenience helper's pins — at minimum add a **Fresnel** row (`ExponentIn`, `BaseReflectFractionIn`, `Normal`). The cost was lighter than #1 (no failed compile — the pin name was needed *before* wiring, not surfaced as a post-wire compile error), but it's the same engine-source detour the table is meant to eliminate. Reinforces that the highest-value edit is completing the pin table, not just adding one node.
- `#3-docs-fix` `IN-REVIEW` developer — Implemented the docs-only minimum (the highest-value, root-cause fix for a discoverability gap), and deliberately did NOT implement the two optional code paths: the `get_material_node_details` per-input expected-type readback rests on a false premise (`UMaterialExpressionNoise` does not override `GetInputType`/`GetInputValueType`, so it returns the base default `MCT_Float` union — the float3 constraint lives in the node's HLSL signature, not its declared pin type — so a readback would report "any float" and reveal nothing; `GetInputType` is also `UE_DEPRECATED(5.6)`), and the compile-error annotation option is speculative and touches the compile path for a narrow payoff. Changes to `Docs/wiki-src/material.authoring.md`: (a) added two rows to the "Common target pins" table — `Noise (add_noise)` → `Position` (float3) / `FilterWidth`, and `Fresnel (add_fresnel)` → `ExponentIn` / `BaseReflectFractionIn` / `Normal` (closing both reported axes: #1 Noise Position type and #2 Fresnel ExponentIn pin name); (b) added a Limitations bullet stating that input pins can require a specific vector width, that `connect_nodes` does not type-check (deferred-type-check — graph correctness is reported by `compile_material`), that the opaque `no matching function for call to 'MaterialExpression…'` overload error is the float2→float3 trap (notably Noise's `Position` float3), and how to fix it (promote a float2 with an `AppendVector` A=float2 B=`Constant` float1 Z). Both edits land in the prelude (above the first `### ` H3) so they render on the `material.authoring` namespace page via `WikiOverlay::LoadGroupPrelude`. Test: added `FWikiHandlerMaterialAuthoringDocumentsPinTypesTest` to `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` — renders the `material.authoring` namespace page via the production `WikiHandler::RenderPage` and asserts on overlay-exclusive markers (`FilterWidth`, `ExponentIn`, `BaseReflectFractionIn`, `float3`, `AppendVector`, `does not type-check`); reverting the table rows or the Limitations bullet drops these markers and fails the assertions. Files: `Docs/wiki-src/material.authoring.md`, `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp`.
- `#4-table-present-but-still-introspected` `IN-REVIEW` reporter — Complementary third axis (PROCESS, clean outcome) from a separate struggle-audit: a stylized-water build authoring `/Game/Materials/M_StylizedWater` (Surface/DefaultLit/Translucent/two-sided; WaterColor/Opacity/PanSpeed params; TexCoord→Panner→WaterNormal.Coordinates UV-pan; BaseColor/Opacity/Normal wired; + `MI_StylizedWater_Shallow` instance), 38 calls, all `ok`, outcome clean. Here the nodes the caller wired are the ones the table **already lists** — Panner (`material.authoring.md` L33 `Coordinate`, `Time`, `Speed`) and TextureSample (L30 `Coordinates, not UVs`) — yet the caller still fired **two `material.authoring.get_material_node_details` reads** ("panner pin names", "WaterNormal pin names") immediately before the `connect_nodes` calls to recover those exact names. Friction note verbatim: *"none - wiki documented every method clearly; I did two get_material_node_details reads to confirm exact pin names (Panner 'Coordinate'/'Speed', TextureSample 'Coordinates') before wiring, which avoided any guesswork or retries."* So #1/#2 are the *omitted-from-table* axis (fixed by #3 by completing the table); this is the *present-in-table-but-not-trusted* axis: the canonical pin names live only in the far-away namespace-prelude table, while the per-method pages the caller actually navigated (`add_panner.md`, `add_texture_sample.md`, `connect_nodes.md` — each generated from the method description; only the prelude references "Common target pins", confirmed `grep -rl` hits `material.authoring.md` alone) carry no pins and don't cross-link the table, so even a careful agent defaults to a live `get_material_node_details` round-trip per node-to-wire. Net cost: 2 avoidable introspection calls on an otherwise first-try build (recurs once per node with non-obvious pins). Proposed downstream docs follow-on (same `docs/wiki-src/material.authoring.md` page, beyond #3's table completion): make the per-method overlay sections for `connect_nodes` / `add_panner` / `add_texture_sample` (and peers) echo or explicitly link the "Common target pins" table so an agent reading a single method page sees the canonical pin vocabulary without a live read — mirrors the `E-graph-standard-exec-pin-names` "publish the fixed vocabulary so connect skips a discovery round-trip" remedy, applied to material pins. Severity stays Medium/Low — purely a recoverable discovery round-trip, never blocks. Deduped into this ticket rather than a new file: same root (material pin-name discoverability), same page, adjacent fix.
- `#5-additional-main-node-display-name-trap` `IN-REVIEW` reporter — Additional evidence, same per-method-page gap (#4 axis) with a concrete **main-node display-name** wrinkle. Stylized-water build authoring `/Game/Fuzz/Materials/M_StylizedWater` (Surface/Translucent/twoSided; WaterTint/RippleSpeed/SurfaceOpacity params; TexCoord→Panner, RippleSpeed→Panner.Speed; WaterTint→BaseColor, SurfaceOpacity→Opacity; compiled clean; + `M_MurkyPond_Inst` instance), outcome done. The caller, working from the per-method `connect_nodes.md` page (which lists NO pins and does not link the "Common target pins" table), reached for the **editor UI display name `"Base Color"` (with a space)** for the main BaseColor pin and got rejected; single retry with `"BaseColor"` succeeded. The error is itself **excellent** (lists the valid pin vocabulary verbatim, so the tool is behaving correctly — NOT a bug, NOT a misleading-error ergo). Replay-confirmed against `/Game/Fuzz/Materials/M_OracleReplay`: `material.authoring.connect_nodes {targetNodeId:"Main", inputName:"Base Color"}` → verbatim `[INVALID_PIN] Unknown input on main node: Base Color. Valid: BaseColor, Metallic, Specular, Roughness, Anisotropy, Normal, Tangent, EmissiveColor, Opacity, OpacityMask, WorldPositionOffset, Displacement, SubsurfaceColor, ClearCoat, ClearCoatRoughness, AmbientOcclusion, Refraction, MaterialAttributes, PixelDepthOffset, ShadingModelFromMaterialExpression, SurfaceThickness, FrontMaterial, CustomizedUVs[0..7]`. Friction note verbatim: *"one connect_nodes failed because I used the display name \"Base Color\" for the main BaseColor pin; the error message listed the valid pin names verbatim (\"BaseColor\") so the single retry succeeded immediately. The wiki for connect_nodes does not document the main-node input names, so I had to learn the exact spelling from the error."* The main-node pins ARE in the namespace-table (`material.authoring.md` L38 `BaseColor, Metallic, …`) — but the table never says these are the *no-space canonical* names rather than the editor's *display* names (`Base Color`, `Emissive Color`), and the `connect_nodes` page the caller navigated carries neither the table nor that caveat. Reinforces the #4 remedy (echo/link the pin table on `connect_nodes.md`) and adds a sub-point: note that main-node inputs use the no-space concatenated names (`BaseColor`, `EmissiveColor`, `OpacityMask`, `WorldPositionOffset`), not the Material Editor's spaced display labels. Cost: 1 failed connect + 1 immediate self-corrected retry on an otherwise first-try build — milder than #4 (the error self-documents), so severity stays Medium/Low.
- `#6-panner-pins-engine-header-read` `IN-REVIEW` reporter — Third corroboration of the #4/#5 per-method-page gap, this time via an **engine-header read** rather than a `get_material_node_details` round-trip. Clean stylized-water build (focus null, namespace material.authoring, outcome clean): authored `/Game/Materials/M_StylizedWater` (Surface/Translucent/DefaultLit/two-sided; WaterColor vector + Opacity/FlowSpeed/Roughness scalar params; TexCoord→Panner with FlowSpeed→Panner.Speed; WaterColor→BaseColor, Opacity→Opacity, Roughness→Roughness; compiled clean compiledWithErrors=false) + `MI_MurkyWater` instance with WaterColor/Opacity overrides, 18 calls, all `ok`, no retries. The Panner pin names the caller wired (`Coordinate`, `Speed`) are **already in the namespace-prelude "Common target pins" table** (`docs/wiki-src/material.authoring.md` L35: `Panner | Coordinate, Time, Speed`, re-confirmed this audit) — yet the caller still read the engine's `MaterialExpressionPanner.h` to recover `Coordinate/Time/Speed` before wiring. Friction note verbatim: *"one engine-source read (MaterialExpressionPanner.h) confirmed the Panner pin names (Coordinate/Time/Speed) but that was optional verification, not a gap."* The caller framed it as optional, but it is the exact off-tool detour the table is meant to eliminate — and it recurred because the per-method pages the caller navigated (`add_panner.md`, `connect_nodes.md`) carry no pins and don't cross-link the table, so the canonical vocabulary in the far-away prelude went untrusted/unseen (same mechanism as #4's two `get_material_node_details` reads and #5's `Base Color` display-name miss). Net cost: 1 avoidable engine-header read on an otherwise first-try build. Reinforces the #4 remedy (echo or link the "Common target pins" table from the per-method `add_panner.md` / `connect_nodes.md` overlay sections so a single-page reader sees the pin vocabulary without an off-tool read) — no new fix, same page (`docs/wiki-src/material.authoring.md`), severity stays Medium/Low (recoverable, never blocked). Deduped into this ticket rather than a new file: same root (material pin-name discoverability via per-method pages), same remedy.
- `#7-additional-non-main-pins-and-loud-error` `IN-REVIEW` reporter — Additional evidence (struggle-audit, clean outcome) corroborating the #4/#5/#6 per-method-page gap with a fresh set of nodes and a second remedy axis. Glowing-sci-fi-panels build authoring `/Game/SciFi/Materials/M_EnergyPanel` (Unlit; GlowColor/GlowIntensity/PanSpeed params; Panner→ComponentMask→Frac scroll modulating GlowColor*GlowIntensity into EmissiveColor), 28 calls, the whole graph/compile/instance run first-try with zero retries. Before wiring its 10 `connect_nodes`, the caller grepped UE engine headers to confirm the **input pin names of the single-input math/util nodes** — Frac and ComponentMask both use `Input`, Multiply/Add use `A`/`B`, Panner uses `Speed` — none of which the per-method `add_math_node.md` / `add_component_mask.md` / `connect_nodes.md` pages enumerate. Friction note verbatim (CallAnalyzer): *"no read surface gives input-pin names, so I grepped the engine to confirm Frac/ComponentMask use 'Input' and Multiply/Add use A/B"* and the agent's own stated motive *"verify the input pin names … so my connects don't silently mis-wire."* Two takeaways: (a) the #4 remedy (echo/link the "Common target pins" table from the per-method `add_*` / `connect_nodes` overlay sections) needs to cover the math/util convenience nodes too — Frac/ComponentMask (`Input`), Multiply/Add/etc. (`A`/`B`) — not just Panner/Texture/Fresnel/Noise; and (b) a NEW remedy axis surfaced by the agent's "silently mis-wire" fear: unlike the **main** node (which returns the excellent `[INVALID_PIN] … Valid: …` list per #5), it is unclear whether `connect_nodes` rejects an unknown inputName on a **non-main** expression node loudly or silently accepts/no-ops it (cf. `B-material-break-connections-named-pin-noop`, the named-pin silent-no-op on the break path). Propose: in addition to the docs echo, make `connect_nodes` return an explicit `[INVALID_PIN]`-style error enumerating the target node's valid input pin names when `inputName` does not match on a non-main node — so a wrong name fails loudly instead of silently mis-wiring, the same self-documenting behavior the main node already has. Net cost this task: 1 off-tool engine-header grep on an otherwise flawless first-try build; same recoverable Medium friction. Deduped here (same root: material pin-name discoverability) rather than a new file.
- `#8-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
