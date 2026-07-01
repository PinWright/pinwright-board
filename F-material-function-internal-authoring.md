---
id: F-material-function-internal-authoring
title: "No way to author a material function's internal graph (add_math_node / connect_nodes / add_expression reject UMaterialFunction)"
status: IN-REVIEW
severity: High
category: feature
tags: [material, material-authoring, material-function, add-math-node, connect-nodes]
---

# Cannot add expression nodes or wire connections inside a UMaterialFunction graph

The `material.authoring` namespace invites you to author a reusable material
function: `create_material_function` documents *"Add inputs/outputs with
add_function_input / add_function_output, then call from materials with
use_material_function."* You can create the asset, add typed inputs, add
outputs, `auto_layout` it (which the wiki says works *"on a UMaterial or a
UMaterialFunction"*), and call it from a material.

But there is **no RPC to build the function's internal logic** — the actual
expression nodes (Multiply, Lerp, etc.) and the wires between the inputs, the
math, and the outputs. Every node-add / connect RPC in both material namespaces
loads its target via the `UMaterial`-only loader (`LOAD_MATERIAL_OR_RETURN` /
bare `LoadObject<UMaterial>`), so handing them a valid `UMaterialFunction` path
fails with `[ASSET_NOT_FOUND] Could not load Material.`:

- `material.authoring.add_math_node` (and every sibling `add_*` convenience node,
  `add_custom_expression`, `connect_nodes`, `disconnect_nodes`, `remove_material_node`)
- `material.graph.add_node` / `add_expression` / `connect_nodes` / `create_nodes` /
  `break_connections` / `remove_node`

So a function authored through this API is a hollow shell: inputs and outputs
exist, but nothing in between can be wired, and the inputs cannot be connected to
the outputs. The function's body is unauthorable through the dedicated node API.

This is the asymmetry: `add_function_input` / `add_function_output` / `auto_layout`
accept a `UMaterialFunction`; the node-add / connect family does not.

**Impact:** The most basic reusable-material-logic task — "make MF_TintBrighten =
BaseColor * Brightness" — cannot be completed with the convenience or graph node
APIs. An attempt agent building exactly this got stuck: `add_math_node` against the
function path returned `[ASSET_NOT_FOUND] Could not load Material.`, and the only
working path was to fall back to the MGIR text-IR (`material.compile_mgir` with an
`entry function` block) — which it could only discover by reading the plugin's C++
loader and the MGIR compiler/decompiler source. There is no discoverable,
node-level route to author function internals.

**Workaround:** Author the function body via `material.compile_mgir` with an
`entry function` block (build the Multiply + its connections + output in the text
IR), then `decompile_mgir` to verify. This works but is undiscoverable from the
`material.authoring` surface and bypasses the typed node API entirely.

**Fix:** Make the node-add / connect / remove RPC family accept a
`UMaterialFunction` target the same way `auto_layout` already does — operate on
`UMaterialFunction::FunctionExpressions` (the function's expression collection)
when the loaded asset is a function, instead of requiring `UMaterial`. The shared
`LOAD_MATERIAL_OR_RETURN` macro (and the `material.graph` bare loaders) would need
a function-aware variant (e.g. resolve to the asset's `UMaterialExpressionCollection`
regardless of whether it is a `UMaterial` or `UMaterialFunction`) so the existing
node/wire mutators can write into either container. Wiring inputs/outputs would
target the function's `FunctionInput`/`FunctionOutput` expressions created by
`add_function_input` / `add_function_output`.

## Repro

1. `material.authoring.create_material_function {name:"MF_OracleReplay", path:"/Game/Materials/Functions"}`
   → `{"assetClass":"MaterialFunction","existsAfter":true}` (success).
2. `material.authoring.add_math_node {assetPath:"/Game/Materials/Functions/MF_OracleReplay", operation:"Multiply", x:0, y:0}`
   → `[ASSET_NOT_FOUND] Could not load Material.`
3. `material.graph.add_expression {assetPath:"/Game/Materials/Functions/MF_OracleReplay", expressionClass:"Multiply", x:0, y:0}`
   → `[ASSET_NOT_FOUND] Could not load Material.` (the low-level sibling also rejects it).

Both confirmed live against the function path created in step 1.

## History
- `#1-initial-repro` `OPEN` reporter — Found while authoring MF_TintBrighten (BaseColor*Brightness). `material.authoring.add_math_node` and `material.graph.add_expression` both reject a valid `UMaterialFunction` path with `[ASSET_NOT_FOUND] Could not load Material.` (replay-confirmed live on a freshly created function). The whole node-add/connect family is `UMaterial`-only via `LOAD_MATERIAL_OR_RETURN`/bare `LoadObject<UMaterial>`, while `add_function_input`/`add_function_output`/`auto_layout` accept the function — so a function's internal logic is unauthorable through the node API. Only workaround is the undiscoverable MGIR `entry function` text-IR path. Not a duplicate: `B-material-function-call-input-name-decoration` (DONE) is about wiring a function-call node's input pins from the *calling material*, not authoring nodes *inside* the function graph.
- `#2-node-level-function-authoring` `IN-REVIEW` developer — GO (all three lenses valid, no objections; defect confirmed present in current source — the node-add/connect/remove family still routed through the `UMaterial`-only loader). Added a function-aware mutation target so the node API authors a function's internal graph the way `auto_layout` already did. **MaterialFinders.h**: generalized `FindExpressionByIdOrName` (shared `ExpressionMatchesNeedle` + a `UMaterialFunction*` overload over `Function->GetExpressions()`); added `FMaterialMutationTarget` (resolves UMaterial-or-UMaterialFunction; exposes Add/Remove/Find-expression, `NotifyEdited`, `AssetObject`) and `LoadMaterialOrFunctionForMutationOrReportError` (tries UMaterial w/ the open-editor guard, then UMaterialFunction, else `UNSUPPORTED_ASSET_CLASS`/`ASSET_NOT_FOUND`, mirroring `auto_layout`'s dispatch). The `FMaterialExpressionFactory::Create(UMaterialFunction*, …)` overloads (write into the function's expression collection) were already present and are now used. Routed the ticket's repro RPCs + core authoring loop through the target: **MaterialAuthoringHandler.cpp** `add_math_node` (new mutation-target `CreateExpressionWithFactoryAndRespond` overload) and **MaterialGraphHandler.cpp** `material.graph.add_node` / `add_expression` / `connect_nodes` / `remove_node`. `connect_nodes` rejects the material-only `Main` pseudo-node on a function with `INVALID_PIN` (wire into a FunctionOutput node instead). Discoverability: new `create_material_function` H3 in `docs/wiki-src/material.authoring.md` documenting the three-phase node-level route; updated handler summaries to say "UMaterial or UMaterialFunction". Regression test **Tests/Assets/TestMaterialFunctionInternalAuthoring.cpp** (`EditorAutomationRpcGateway.Material.Authoring.FunctionInternalAuthoring`) drives the real handlers against a persisted function: create → add input/output → `add_math_node` Multiply → `add_expression` Constant → wire input→Multiply.A→FunctionOutput.A → assert the function's own ExpressionCollection gained the nodes and the wires landed → `Main`-on-function rejected → `remove_node` drops the node. Pre-fix the add RPCs errored on the function path, so the success + membership assertions fail if reverted. Not compiled/tested here (later phase). Convenience adds beyond `add_math_node` (`add_world_position` etc.) and `material.graph.add_texture_sample`/`create_nodes` were left on the material-only path as out of scope for the repro/core-loop; a follow-up can widen them via the same target.
</content>
</invoke>
