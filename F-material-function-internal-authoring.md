---
id: F-material-function-internal-authoring
title: "No way to author a material function's internal graph (add_math_node / connect_nodes / add_expression reject UMaterialFunction)"
status: IN-REVIEW
severity: High
category: feature
tags: [material, material-authoring, material-function, add-math-node, connect-nodes]
encounters: 2
lastSeen: 2026-07-01T17:01:55.7647647+03:00
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
- `#3-additional-authoring-connect-still-broken` `IN-REVIEW` reporter — Additional evidence (fix-gap angle): at HEAD the IN-REVIEW fix widened `material.graph.connect_nodes` (verified working on a function by the attempt), but the parallel authoring-namespace sibling **`material.authoring.connect_nodes` was left on the UMaterial-only path** and still rejects a valid `UMaterialFunction`. Replay: `create_material_function {name:"MF_OracleReplayAuthConnect", path:"/Game/Materials/Functions"}` → success (`assetClass:"MaterialFunction"`), then `material.authoring.connect_nodes {assetPath:"/Game/Materials/Functions/MF_OracleReplayAuthConnect", sourceNodeId:"SomeNode", targetNodeId:"OtherNode", inputName:"A"}` → `[ASSET_NOT_FOUND] Could not load Material.` — the loader fails before any node lookup. Guilty line: `MaterialAuthoringHandler.cpp:1422` `LOAD_MATERIAL_OR_RETURN();` → `MaterialFinders.h:98` `Ctx.SendError(TEXT("ASSET_NOT_FOUND"), TEXT("Could not load Material."));`. The whole authoring-namespace connect/remove family is still UMaterial-only: `disconnect_nodes` (1486), `remove_material_node` (bare `LoadObject<UMaterial>` at 2993 → error at 2996), `add_custom_expression` (1325). Impact for the attempt agent: it reached for `material.authoring.connect_nodes` first (it sits in the authoring namespace next to `add_function_input`/`add_function_output` it had just used), hit ASSET_NOT_FOUND 4×, and only recovered by re-reading the wiki and switching to `material.graph.connect_nodes`. So the fix as scoped in `#2` leaves the more discoverable authoring-namespace wiring verb broken — route it (and its disconnect/remove siblings) through the same `FMaterialMutationTarget` before DONE, or the asymmetry the ticket is about persists in the authoring namespace.
- `#2-node-level-function-authoring` `IN-REVIEW` developer — GO (all three lenses valid, no objections; defect confirmed present in current source — the node-add/connect/remove family still routed through the `UMaterial`-only loader). Added a function-aware mutation target so the node API authors a function's internal graph the way `auto_layout` already did. **MaterialFinders.h**: generalized `FindExpressionByIdOrName` (shared `ExpressionMatchesNeedle` + a `UMaterialFunction*` overload over `Function->GetExpressions()`); added `FMaterialMutationTarget` (resolves UMaterial-or-UMaterialFunction; exposes Add/Remove/Find-expression, `NotifyEdited`, `AssetObject`) and `LoadMaterialOrFunctionForMutationOrReportError` (tries UMaterial w/ the open-editor guard, then UMaterialFunction, else `UNSUPPORTED_ASSET_CLASS`/`ASSET_NOT_FOUND`, mirroring `auto_layout`'s dispatch). The `FMaterialExpressionFactory::Create(UMaterialFunction*, …)` overloads (write into the function's expression collection) were already present and are now used. Routed the ticket's repro RPCs + core authoring loop through the target: **MaterialAuthoringHandler.cpp** `add_math_node` (new mutation-target `CreateExpressionWithFactoryAndRespond` overload) and **MaterialGraphHandler.cpp** `material.graph.add_node` / `add_expression` / `connect_nodes` / `remove_node`. `connect_nodes` rejects the material-only `Main` pseudo-node on a function with `INVALID_PIN` (wire into a FunctionOutput node instead). Discoverability: new `create_material_function` H3 in `docs/wiki-src/material.authoring.md` documenting the three-phase node-level route; updated handler summaries to say "UMaterial or UMaterialFunction". Regression test **Tests/Assets/TestMaterialFunctionInternalAuthoring.cpp** (`EditorAutomationRpcGateway.Material.Authoring.FunctionInternalAuthoring`) drives the real handlers against a persisted function: create → add input/output → `add_math_node` Multiply → `add_expression` Constant → wire input→Multiply.A→FunctionOutput.A → assert the function's own ExpressionCollection gained the nodes and the wires landed → `Main`-on-function rejected → `remove_node` drops the node. Pre-fix the add RPCs errored on the function path, so the success + membership assertions fail if reverted. Not compiled/tested here (later phase). Convenience adds beyond `add_math_node` (`add_world_position` etc.) and `material.graph.add_texture_sample`/`create_nodes` were left on the material-only path as out of scope for the repro/core-loop; a follow-up can widen them via the same target.
- `#4-additional-function-output-input-name` `IN-REVIEW` reporter — Additional evidence (PROCESS/discoverability angle, distinct from #3's authoring-connect breakage). Once the function body is wireable (via `material.graph.connect_nodes` per #3's workaround), feeding a source INTO the `UMaterialExpressionFunctionOutput` node that `add_function_output` creates needs an `inputName`, but that node's single input pin is **unnamed** (`GetInputName` returns `NAME_None` — confirmed in engine `MaterialExpressionFunctionOutput.h`), so there is no discoverable pin name. In this MF_TintUtil task (focus `material.authoring.add_function_output`, outcome done — the two outputs themselves worked, distinct output indices, clean compile) the attempt agent had to dive into engine source to wire the outputs: Bash grep (wrong path, exit 2) -> Glob (No files found) -> Bash find -> `Read` of `C:/UE_5.7/.../MaterialExpressionFunctionOutput.h`, then empirically guessed `inputName:"A"` (the `FExpressionInput` member name) and it worked — 3 filesystem/Bash discovery calls + 1 engine-header Read before the first successful FunctionOutput wire. Friction note verbatim: *"had to check engine source (MaterialExpressionFunctionOutput.h) to learn the FunctionOutput input pin returns NAME_None yet matches inputName 'A' ... a small discoverability gap since the required inputName for a FunctionOutput target is undocumented."* Remedy (lands on the same `docs/wiki-src/material.authoring.md` `create_material_function` H3 that this ticket's #2 fix already adds, so it is a one-line addition there): (a) code — make `inputName` OPTIONAL on `material.graph.connect_nodes` when the target node has exactly one input pin (a FunctionOutput always does), defaulting to that pin; and/or (b) docs — state on the H3 / `add_function_output` page that to feed a FunctionOutput you pass its node id as `targetNodeId` with `inputName:"A"` (or that `inputName` may be omitted). Recorded here (not a new file) because it is the sibling discoverability gap on the exact function-authoring path this ticket enables, fixable in the same H3; the ticket's own High severity (missing capability) is unchanged. severity rationale: impact=docs/discoverability (engine-source dive, known 'A' workaround) x reach=rare (function-internal authoring) -> Low.
