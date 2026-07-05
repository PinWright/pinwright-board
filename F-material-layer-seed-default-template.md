---
id: F-material-layer-seed-default-template
title: "create_material_layer / create_material_layer_blend produce EMPTY function bodies (no default template) — freshly created layer/blend fails material validation until hand-authored"
status: OPEN
severity: Medium
category: feature
tags: [material, material-authoring, material-layers, factory-no-default-template]
encounters: 1
lastSeen: 2026-07-05T13:04:36.5063694+03:00
---

# Layer / blend factory RPCs skip the default template body, so a new layer stack never compiles as-created

`material.authoring.create_material_layer` and
`material.authoring.create_material_layer_blend` create the correct concrete
asset classes (`UMaterialFunctionMaterialLayer` /
`UMaterialFunctionMaterialLayerBlend`) but leave the function **body empty** —
no inputs, no output, no default expression graph. UE's own content-browser
factory for these asset types seeds a validation-passing template: a blend
comes pre-wired with two `MaterialAttributes` inputs (Bottom/Top) feeding a
`BlendMaterialAttributes` into the output, and a layer comes with a
`MakeMaterialAttributes` (or `Set Material Attributes`) wired to its output.

Because the RPC-created assets are hollow, the moment you drop them into a
material's layer stack (via the working `set_material_layer_stack`) and compile,
the material fails validation — not with a friendly "your blend has no body"
but with the raw engine validators:

- 1st compile: `(Node MaterialAttributeLayers) Blend 0, MLB_RockMoss, must have two MaterialAttributes inputs.`
- 2nd compile (after adding the two blend inputs): `(Function MLB_RockMoss) Missing function input 'BottomLayer' / 'TopLayer'` — the layer functions themselves still have no output, so the blend's inputs resolve to nothing.

There is no single-call or template option to get a **ready-to-compile** default
layer/blend. The caller must hand-author every body before the stack will
compile: for the blend, add two function inputs + a `BlendMaterialAttributes` +
an alpha `ScalarParameter` + a function output + 4 wires; for each layer, add a
`VectorParameter` + `MakeMaterialAttributes` + a function output + 2 wires. That
is ~25 authoring calls and **two failed compile round-trips** to reach the state
the editor factory hands you for free on asset creation.

**Why it matters.** The whole point of `F-material-layers-asset-authoring`
(which added these three RPCs) is to author a custom layer stack end-to-end via
automation. Producing empty bodies means the common case — "rock base layer +
moss layer blended on top" — cannot be created-and-compiled without the caller
reverse-engineering UE's material-layer validation rules from engine source. In
this task the agent had to read `BlendMaterialAttributes.h` (to learn A/B/Alpha
are all required), `MaterialExpressions.cpp` (the `must have two MA inputs` /
`layer <= 1 input` validators), and `MaterialExpressionFunctionOutput.h` (to
learn the FunctionOutput input pin is `A`) just to hand-build what a template
would have seeded.

**Distinct from `F-material-function-internal-authoring`** (the ticket the judge
filed for this task): that ticket is about the *node-add / connect API rejecting
a `UMaterialFunction` path* (`add_material_node` / `add_scalar_parameter` →
`[ASSET_NOT_FOUND] Could not load Material.`, workaround = `material.graph.add_expression`).
Even once that is fixed and function bodies are freely authorable, the caller
*still* has to author every layer/blend body by hand for the standard case. This
ticket asks the **factory RPCs to seed the validation-passing default template**
so the standard stack compiles as-created — a different root cause and a
different fix from the node-API rejection.

**Workaround:** After `create_material_layer` / `create_material_layer_blend`,
hand-author each function body (inputs, `BlendMaterialAttributes` / `MakeMaterialAttributes`,
alpha param, output, wiring) via `material.graph.add_expression` +
`material.graph.connect_nodes` before compiling — ~25 extra calls across the two
layers + one blend, discovered only after two failed compiles.

**Fix:** Have `create_material_layer` / `create_material_layer_blend` seed the
same default template body UE's `UMaterialFunctionMaterialLayerFactory` /
`UMaterialFunctionMaterialLayerBlendFactory` content-browser path produces (the
two-input `BlendMaterialAttributes` blend; the `MakeMaterialAttributes` layer),
or add an opt-in `seedTemplate` / `template` param (default on) that does so. A
freshly created layer + blend should drop into a `set_material_layer_stack` and
pass `compile_material` on the first try.

severity rationale: impact=soft-blocker (standard layered-material task needs ~25 hand-authoring calls + two failed compiles + multi-header engine source dive to reconstruct the validator template) x reach=rare (material-layer authoring path) -> Medium (the source-dive-and-failed-compile cost holds it at Medium rather than Low; these RPCs exist specifically to make this workflow automatable).

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a layered-terrain material task (focus `material.authoring.set_material_layer_stack`; outcome clean after hand-authoring). `create_material_layer` (ML_TerrainRock, ML_TerrainMoss) and `create_material_layer_blend` (MLB_RockMoss) each produced an EMPTY function body; `set_material_layer_stack` populated the stack correctly (Layers=[rock,moss] Blends=[MLB]) but the 1st `compile_material` failed `(Node MaterialAttributeLayers) Blend 0, MLB_RockMoss, must have two MaterialAttributes inputs.` and the 2nd failed `(Function MLB_RockMoss) Missing function input 'BottomLayer' / 'TopLayer'` — both because the factory RPCs seeded no template. Only the 3rd compile (after hand-authoring both layer bodies + the blend body: ~25 authoring calls + engine-source dives into `BlendMaterialAttributes.h`, `MaterialExpressions.cpp` validators, `MaterialExpressionFunctionOutput.h`) came back clean. CallAnalyzer independently flagged `material.authoring.create_material_layer_blend` as "surprising". Friction note verbatim: *"create_material_layer and create_material_layer_blend produce EMPTY function assets (no default template), unlike the UE content-browser factory which seeds Bottom/Top BlendMaterialAttributes and a MakeMaterialAttributes layer template — so the first compile failed layer validation ... forcing me to hand-author both layer bodies and the blend body ... before it compiled."* Distinct root cause from `F-material-function-internal-authoring` (node-API rejects function paths) and a follow-up limitation of the RPCs added in `F-material-layers-asset-authoring` (DONE).
