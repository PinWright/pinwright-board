---
id: F-material-layer-seed-default-template
title: "create_material_layer / create_material_layer_blend produce EMPTY function bodies (no default template) — freshly created layer/blend fails material validation until hand-authored"
status: IN-REVIEW
severity: Medium
category: feature
tags: [material, material-authoring, material-layers, seed-default-template]
encounters: 1
lastSeen: 2026-07-05T13:04:36.5063694+03:00
---

# Layer / blend create RPCs skip the default template body, so a new layer stack never compiles as-created

`material.authoring.create_material_layer` and
`material.authoring.create_material_layer_blend` create the correct concrete
asset classes (`UMaterialFunctionMaterialLayer` /
`UMaterialFunctionMaterialLayerBlend`) but leave the function **body empty** —
no inputs, no output, no default expression graph.

**Where the template actually comes from (ticket-reword correction).** The
original report attributed the default template to the engine content-browser
**factory**. That is wrong: `UMaterialFunctionMaterialLayerFactory::FactoryCreateNew`
/ `...BlendFactory::FactoryCreateNew` (`EditorFactories.cpp:571-600`) only
`NewObject()` + `SetMaterialFunctionUsage(...)` — they seed **nothing**, exactly
like these RPCs. The validation-passing template is seeded by the material-function
**editor on-open** (`FMaterialEditor`, `MaterialEditor.cpp:743-841`) when the
function is empty: an output node plus input pins always, and the *wired*
`SetMaterialAttributes` / `BlendMaterialAttributes` cluster as UE's "example graph"
(gated in the editor behind the `bExampleLayersAndBlends` experimental setting).
The RPC path never opens the editor, so it gets none of that. The real seeded
shapes are:
- **Layer:** one `MaterialAttributes` `FunctionInput` (name "Material Attributes",
  `bUsePreviewValueAsDefault=true`) → `SetMaterialAttributes` → a
  `UMaterialExpressionMaterialLayerOutput` (a `FunctionOutput` subclass). NOT a
  `MakeMaterialAttributes` (the original report's guess).
- **Blend:** two `MaterialAttributes` `FunctionInput`s named "Top Layer" /
  "Bottom Layer" (with `BlendInputRelevance` Top/Bottom, `bUsePreviewValueAsDefault=true`)
  → `BlendMaterialAttributes` (A=Bottom, B=Top) → `MaterialLayerOutput`. There is
  **no** alpha `ScalarParameter` (the original report invented one).

Because the RPC-created assets are hollow, the moment you drop them into a
material's layer stack (via the working `set_material_layer_stack`) and compile,
the material fails validation — not with a friendly "your blend has no body"
but with the raw engine validators:

- 1st compile: `(Node MaterialAttributeLayers) Blend 0, MLB_RockMoss, must have two MaterialAttributes inputs.` (`MaterialExpressions.cpp:7399` counts `FunctionInput`s == 2)
- 2nd compile (after adding the two blend inputs): `(Function MLB_RockMoss) Missing function input 'BottomLayer' / 'TopLayer'` — an unconnected `FunctionInput` without `bUsePreviewValueAsDefault` compiles to this error (`MaterialExpressions.cpp:16362`).

There is no single-call or template option to get a **ready-to-compile** default
layer/blend. The caller must hand-author every body before the stack will
compile: for the blend, two function inputs + a `BlendMaterialAttributes` + a
function output + wires; for each layer, a `SetMaterialAttributes` + a function
output + wires. That is ~25 authoring calls and **two failed compile round-trips**
to reach the state the editor hands you for free on asset open.

**Why it matters.** The whole point of `F-material-layers-asset-authoring`
(which added these three RPCs) is to author a custom layer stack end-to-end via
automation. Producing empty bodies means the common case — "rock base layer +
moss layer blended on top" — cannot be created-and-compiled without the caller
reverse-engineering UE's material-layer validation rules from engine source.

**Distinct from `F-material-function-internal-authoring`.** That ticket is about
the *node-add / connect API rejecting a `UMaterialFunction` path*
(`add_material_node` / `add_scalar_parameter` → `[ASSET_NOT_FOUND]`, workaround =
`material.graph.add_expression`). Even once that is fixed and function bodies are
freely authorable, the caller *still* has to author every layer/blend body by
hand for the standard case. This ticket is a different root cause (create verbs
seed no template) and a different fix (seed the template in the create handler,
replicating the editor's on-open seeding — the factory cannot help).

**Workaround:** After `create_material_layer` / `create_material_layer_blend`,
hand-author each function body (inputs, `SetMaterialAttributes` /
`BlendMaterialAttributes`, output, wiring) via `material.graph.add_expression` +
`material.graph.connect_nodes` before compiling — ~25 extra calls, discovered
only after two failed compiles.

**Fix (rewored scope):** Add an opt-in `seedTemplate` boolean param (**default on**)
to `create_material_layer` / `create_material_layer_blend` that seeds the
validation-passing default template the material-function editor produces on-open,
built directly in the create handler (NOT via the factory, which seeds nothing):
- layer → `MaterialAttributes` `FunctionInput` → `SetMaterialAttributes` → `MaterialLayerOutput`;
- blend → two `MaterialAttributes` `FunctionInput`s (Top/Bottom, with `BlendInputRelevance`) → `BlendMaterialAttributes` → `MaterialLayerOutput`.
A freshly created layer + blend should drop into a `set_material_layer_stack` and
pass `compile_material` on the first try. Keep the empty-body path reachable via
`seedTemplate:false` so the custom-authoring route (per
`F-material-function-internal-authoring`) is not forced to strip a pre-seeded cluster.

severity rationale: impact=soft-blocker (standard layered-material task needs ~25 hand-authoring calls + two failed compiles + multi-header engine source dive to reconstruct the validator template) x reach=rare (material-layer authoring path) -> Medium (the source-dive-and-failed-compile cost holds it at Medium rather than Low; these RPCs exist specifically to make this workflow automatable).

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a layered-terrain material task (focus `material.authoring.set_material_layer_stack`; outcome clean after hand-authoring). `create_material_layer` (ML_TerrainRock, ML_TerrainMoss) and `create_material_layer_blend` (MLB_RockMoss) each produced an EMPTY function body; `set_material_layer_stack` populated the stack correctly (Layers=[rock,moss] Blends=[MLB]) but the 1st `compile_material` failed `(Node MaterialAttributeLayers) Blend 0, MLB_RockMoss, must have two MaterialAttributes inputs.` and the 2nd failed `(Function MLB_RockMoss) Missing function input 'BottomLayer' / 'TopLayer'` — both because the create RPCs seeded no template. Only the 3rd compile (after hand-authoring both layer bodies + the blend body: ~25 authoring calls + engine-source dives into `BlendMaterialAttributes.h`, `MaterialExpressions.cpp` validators, `MaterialExpressionFunctionOutput.h`) came back clean. CallAnalyzer independently flagged `material.authoring.create_material_layer_blend` as "surprising". Friction note verbatim: *"create_material_layer and create_material_layer_blend produce EMPTY function assets (no default template), unlike the UE content-browser factory which seeds Bottom/Top BlendMaterialAttributes and a MakeMaterialAttributes layer template — so the first compile failed layer validation ... forcing me to hand-author both layer bodies and the blend body ... before it compiled."* Distinct root cause from `F-material-function-internal-authoring` (node-API rejects function paths) and a follow-up limitation of the RPCs added in `F-material-layers-asset-authoring` (DONE).
- `#2-reword` `IN-REVIEW` fuzz2 — REWORD then implement. Verified against engine source: the content-browser **factory** seeds nothing (`EditorFactories.cpp:571-600` = `NewObject` + `SetMaterialFunctionUsage` only); the template is seeded by `FMaterialEditor` **on-open** (`MaterialEditor.cpp:743-841`), layer = `SetMaterialAttributes` (not the report's `MakeMaterialAttributes`), blend = two `MaterialAttributes` inputs "Top Layer"/"Bottom Layer" → `BlendMaterialAttributes` (no alpha `ScalarParameter`). Corrected the ticket's mechanism + node names. Intended scope: opt-in `seedTemplate` bool (default on) on both create verbs that seeds that compile-ready template in the create handler; `seedTemplate:false` keeps the empty-body path for custom authoring. Category `feature`, severity `Medium` unchanged. Shipped in `Handlers/Material/MaterialAuthoringHandler.cpp`: file-local `SeedMaterialLayerFunctionTemplate` helper + `seedTemplate` `RPC_PARAM_DEF` (default `"true"`) on `create_material_layer` and `create_material_layer_blend`; seeds `MaterialLayerOutput` + `SetMaterialAttributes` (layer) / two "Top Layer"/"Bottom Layer" `FunctionInput`s → `BlendMaterialAttributes` (blend), wired via `UMaterialEditingLibrary::ConnectMaterialExpressions`. Regression test = adopted red test `PinWright.Material.Authoring.LayerSeedTemplate` (`Tests/Assets/TestMaterialLayerSeedTemplate.cpp`): observed failing pre-fix (empty body), passes post-fix; plugin builds clean.
