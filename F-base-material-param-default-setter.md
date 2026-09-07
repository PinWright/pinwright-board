---
id: F-base-material-param-default-setter
title: "No RPC to edit a base UMaterial's scalar/vector parameter default value in place (typed setters are instance-only)"
status: OPEN
severity: Low
category: feature
tags: [material, material-authoring, base-material-param-default, parameters, coverage-gap]
encounters: 2
costly: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# No in-place setter for a base UMaterial's scalar/vector parameter default

The typed material parameter setters — `material.authoring.set_scalar_parameter_value`,
`set_vector_parameter_value`, `set_texture_parameter_value` — all operate on a
`UMaterialInstanceConstant` (their wiki blurbs say "Override a named scalar
parameter on a UMaterialInstanceConstant"), and `set_material_parameter` is a
NOT_IMPLEMENTED stub (see `B-material-stub-handlers-silent-success`). There is
**no** RPC that mutates a base `UMaterial`'s scalar/vector parameter **default
value** (the `UMaterialExpressionScalarParameter::DefaultValue` /
`UMaterialExpressionVectorParameter::DefaultValue` that the parameter expression
node actually stores). `get_material_info` surfaces these defaults (Metal Colour,
Rough 1, Rough 2, Colour in this task), and `add_scalar_parameter` /
`add_vector_parameter` can set a default at CREATE time, but nothing edits an
existing parameter node's default afterward.

So an agent that wants to change the "look" of an existing base material by
tweaking a color/roughness parameter default has no typed path. The full
material-graph surface (`material.graph.add_node`/`connect_nodes`/`remove_node`,
`material.authoring.add_*`) can only add/rewire/remove nodes — there is no
`material.graph.set_node_property` / `set_expression_property` to edit an existing
node's UPROPERTY (confirmed: the material.graph verbs are add_node, create_nodes,
connect_nodes, get_node_details, remove_node — no property setter).

## Workaround

- `set_two_sided` (a documented base-material render-flag toggle) — a *different*
  kind of look edit; does not touch parameter defaults. Used cleanly one-shot in
  this task as the stand-in "look" change.
- Remove the parameter node (`remove_material_node`) and re-add it with the new
  default (`add_scalar_parameter`) — heavy: loses any existing wiring to that node.

## What it should do

A `material.authoring.set_scalar_parameter_default` /
`set_vector_parameter_default` (assetPath, parameterName, value, save?) that finds
the parameter expression node on the base `UMaterial`, writes its `DefaultValue`,
calls `PostEditChangeProperty` + recompiles, and saves — or extend the existing
typed setters to accept a base `UMaterial` and route to the expression node's
`DefaultValue` instead of an instance override.

## Evidence

Clean `source_control.revert` task (focus source_control.revert, outcome
tool_bug for a separate revert defect). Prep already anticipated the instance-only
setter limitation and pre-planned the `set_two_sided` fallback. In the attempt,
`get_material_info` reported M_Metal / M_Tile as base UMaterials carrying
scalar/vector parameters with defaultValues; a whole-wiki case-insensitive grep
for parameter-default setters plus a `material*.md` filename glob turned up no
base-material parameter-default editor (CallAnalyzer inefficiency, pattern
"missing"). Attempt SAY: "The typed scalar/vector setters target material
instances, not base materials." The agent fell back to `set_two_sided` as the
look edit rather than tweaking the actual color/roughness parameter defaults.

Distinct from `F-material-instance-overrides-incomplete` (the
UMaterialInstanceConstant override surface — different UE class + mechanism) and
from `E-get-material-info-no-param-defaults` (the READ side — get_material_info
omitting the default field). This is the base-material default WRITE gap.

severity rationale: impact=soft-blocker (a base-material default edit is doable
only via a heavy remove+re-add node workaround, and the broader "adjust the look"
goal has a clean one-shot path via set_two_sided) × reach=rare (base-material
parameter-default editing is far less common than instance overrides) -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced as a capability gap in a clean-outcome source_control.revert task (the task's tool_bug was a separate revert defect, filed as B-source-control-revert-no-package-reload). CallAnalyzer inefficiency (pattern "missing", suspect `material.authoring.set_scalar_parameter_value`): the typed scalar/vector/texture setters are UMaterialInstanceConstant-only and `set_material_parameter` is a NOT_IMPLEMENTED stub, so there is no way to edit a base UMaterial's scalar/vector parameter DEFAULT value in place. Confirmed against the plugin method list: material.graph exposes only add_node/create_nodes/connect_nodes/get_node_details/remove_node (no existing-node property setter), and material.authoring add_scalar/vector_parameter set a default only at create time. Agent worked around it with `set_two_sided` (a documented base-material render-flag toggle) in one shot. Propose a `set_scalar/vector_parameter_default` RPC (or extend the typed setters to accept a base UMaterial and mutate the expression node's DefaultValue + recompile). Low — clean one-shot completion via the set_two_sided fallback; distinct from the instance-override gaps in F-material-instance-overrides-incomplete and the read-side gap in E-get-material-info-no-param-defaults.
- `#2-measured-error-and-property-set-workaround` `OPEN` reporter — Second encounter, adding two things `#1` did not have. **(1) The measured failure, not an inference.** `#1` established instance-only from wiki blurbs and the method list; this encounter hit it: `material.authoring.set_vector_parameter_value` and `set_scalar_parameter_value` pointed at a base `UMaterial` answer `ASSET_NOT_FOUND: Could not load material instance`. Re-derived at HEAD in this checkout — all three typed setters do a bare `LoadObject<UMaterialInstanceConstant>` and treat a null as missing: scalar `MaterialAuthoringHandler.cpp:2127` load / `:2130` error, vector `:2166` / `:2168`, texture `:2217` / `:2219`. **That error code is itself wrong**: the asset exists and loads fine, it is simply the wrong class, and `ASSET_NOT_FOUND` sends the caller off to re-check a path that was correct — which is why the gap reads as a broken path rather than as this ticket's missing capability, and part of why it took an audit to find. `UNSUPPORTED_ASSET_CLASS` is the spelling this handler family already uses for exactly this discrimination, and the correct helper is **in the same file, 80 lines below**: `LoadMaterialInstanceOrError` (`:2251-2272`) returns `UNSUPPORTED_ASSET_CLASS` naming the class actually received and falls back to `ASSET_NOT_FOUND` only when nothing loads — and six other handlers already call it (`:2290`, `:2402`, `:2443`, `:2483`, `:2612`, `:2749`). `MaterialFinders.h:107-111` names this precise anti-pattern in its own comment ("Both used to answer ASSET_NOT_FOUND, which sends the caller off to re-check a path that was correct all along — the most expensive wrong error in this family, because a material INSTANCE is the single most common thing to point one of these verbs at"), and `F-material-instance-overrides-incomplete` records the mirror-image case working correctly (`get_material_info` rejecting a `UMaterialInstanceConstant` with `UNSUPPORTED_ASSET_CLASS`, its `#2` citing `:2147`). So the family fixed the UMaterial-expecting direction and left the three UMaterialInstanceConstant-expecting typed setters behind. **Decision: filed separately as `B-material-param-setters-wrong-class-error` rather than as a line here** — it is a different defect (a misleading error code on a normal path, three lines to fix by swapping the loads for the existing helper) from this ticket's missing capability, it is worth fixing whether or not this feature ever ships, and folding a mechanical bug into a Low feature request would make this an umbrella and tie the bug's fix to the feature's schedule. Cross-linked both ways. **(2) A working workaround `#1` does not know about, strictly better than the remove-and-re-add route above**: `property.set` (`UtilityPropertyHandler.cpp:1020` — "works on actors, components, asset CDOs, and arbitrary UObjects") against the parameter expression object itself, `propertyName: "DefaultValue"`, which is a real UPROPERTY on both classes (`Engine/Public/Materials/MaterialExpressionVectorParameter.h:18`, `FLinearColor`; `MaterialExpressionScalarParameter.h:29`, `float`). Non-destructive: the node keeps its GUID, its name and all its wiring, unlike remove + re-add. **`#1`'s conclusion was right about `material.graph` and wrong about the plugin** — it surveyed the `material.graph` namespace, correctly found no `set_node_property` there, and concluded no node UPROPERTY could be edited; the generic reflection setter lives in the `property` namespace and was outside the survey. **Discoverability cost, which is the real price of this workaround**: `property.set` needs the expression's object path, ending in `MaterialExpressionVectorParameter_N`, and that name is reachable through exactly one field — `nodeName` (`Expr->GetName()`) emitted by `MGIRExpressionUtils::BuildExpressionDetailsJson` at `MGIRExpressionUtils.h:205`. That builder runs only on `material.graph.get_node_details`' single-node branch (`MaterialGraphHandler.cpp:367`, requires a resolved `nodeId`); the list-all branch (`:377-390`) emits only `nodeId` (a GUID), `nodeType`, `index` and `desc` — **no object name**. So the sequence is: `get_node_details` with no `nodeId` to collect GUIDs, then one `get_node_details` per candidate to read `nodeName`, then `property.set`. N+2 calls to change one number, and nothing documents the route. Adjacent to `E-material-expression-value-property-undocumented` (OPEN, Low) — same class of cost, different field: that ticket is about literal nodes' *value* keys (`Constant3Vector.Constant`, `Constant.R`) being unguessable from the `material.graph` overlay, this is about the parameter node's *object name* being unobtainable from the list-all readback. Its `#1` names only `list_expression_types` / `search_expression_types` and never mentions `get_node_details`, so it does not already cover this. Severity unchanged at Low and status unchanged at OPEN: the better workaround, if anything, reinforces the existing "soft blocker, rare reach" rationale rather than raising it — but it is non-obvious enough that it is worth writing down, and the wrong error code above is what makes finding it harder than it should be.
