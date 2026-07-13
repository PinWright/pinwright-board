---
id: F-base-material-param-default-setter
title: "No RPC to edit a base UMaterial's scalar/vector parameter default value in place (typed setters are instance-only)"
status: OPEN
severity: Low
category: feature
tags: [material, material-authoring, base-material-param-default, parameters, coverage-gap]
encounters: 1
lastSeen: 2026-07-13T11:52:31.7378932+03:00
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
