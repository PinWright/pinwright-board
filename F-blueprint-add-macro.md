---
id: F-blueprint-add-macro
title: "No imperative `blueprint.add_macro` mirroring `blueprint.add_function`"
status: DONE
severity: Medium
category: feature
tags: [blueprint, macro, mcp-rpc, api-symmetry, bpir]
---

# Missing: imperative macro creation RPC

`blueprint.add_function` (in `BlueprintFunctionHandler.cpp`) lets callers
create an empty UFunction graph with caller-specified input/output pin
signature, then iteratively populate the body via
`blueprint.graph.create_node` calls scoped to that graph. Macros have no
equivalent verb: the only way to create a macro graph through the plugin
is `blueprint.compile_bpir` with an `entry macro Name(...) { ... }` block,
which forces the caller to declare the entire body as BPIR text in a single
bulk operation.

This is the same imperative-vs-bulk asymmetry that
`F-create-node-collapse-target-param` (DONE) closed at the node-vocabulary
layer, only one level up — at the graph-creation layer. Callers writing
mixed automation flows (procedural codegen, refactor passes, partial-body
authoring with later edits) currently must:
1. Compose the macro body upfront as a BPIR text blob, OR
2. Compile a no-op `entry macro Foo() {}` shell via `compile_bpir`, then
   discover the auto-named tunnel nodes and populate the body via
   `blueprint.graph.create_node` scoped to the new macro graph.

Path (2) works (verified by the `F-remove-macro` `#3-verified-macro-removal`
tester note, which used `entry macro TestMacro() {}` to create a 2-tunnel
shell), but it requires the caller to keep a working BPIR compiler in the
mental model just to produce an empty graph. There is no symmetric verb
that says "give me a macro graph with this exact tunnel-pin signature."

**Proposed:** `blueprint.add_macro(blueprintPath, name, inputs:[...],
outputs:[...], execExits:[...])` — creates a new `UEdGraph` in
`Blueprint->MacroGraphs` with paired `UK2Node_Tunnel` entry/exit nodes
matching the requested pin signature, supports pure (no exec) and
multi-exit forms, returns `{ graphName, entryNodeId, exitNodeId }` usable
as the `graphName` scope for subsequent `blueprint.graph.create_node`
calls. Tunnel-pin signature param shape mirrors `add_function`'s
`inputs` / `outputs` for caller vocabulary parity.

**Out of scope:** macro instance call-site generation (already handled by
`blueprint.graph.create_node` with the macro graph as the target — same as
how `add_function` doesn't generate call sites either).

**Files affected (sketch):**
- `Plugins\EditorAutomationRpcGateway\Source\EditorAutomationRpcGateway\Private\Handlers\Blueprint\BlueprintFunctionHandler.cpp` (new `add_macro` handler alongside `add_function`, or split into a sibling file if the function grows)
- `Plugins\EditorAutomationRpcGateway\Source\EditorAutomationRpcGatewayTests\Private\Blueprint\TestBlueprintHandlers.cpp` (regression: pure macro, multi-exit macro, scope-by-graphName for follow-up `create_node`)
- `Plugins\EditorAutomationRpcGateway\docs\wiki\blueprint.md` (new section)

**Related:** `F-create-node-collapse-target-param` (DONE — closed the same
asymmetry one layer below at node vocabulary). `F-remove-macro` (DONE —
added the symmetric remove path; this ticket adds the symmetric create
path).

## History
- `#1-initial-feature-request` `OPEN` reporter — Confirmed via grep: `BlueprintFunctionHandler.cpp:65` registers `blueprint.add_function` with explicit "follow with blueprint.compile_bpir or blueprint.graph.create_node calls scoped to the new graph" wording in its summary, but no corresponding `blueprint.add_macro` exists anywhere in `Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/`. `rpc-method-reference.generated.md` lists `blueprint.add_function` (line 2976) and material-domain `add_function_input/output`, but no macro creation verb. Macro graphs are read/walked in many handlers (`BlueprintGraphHandler.cpp:847`, `BlueprintComponentHandler.cpp:811`, `BlueprintVariableCleanupHandler.cpp:30`) but never created outside the BPIR compiler path. Same imperative-vs-bulk asymmetry that `F-create-node-collapse-target-param` closed at the node-vocabulary layer.
- `#2-add-macro-rpc` `IN-REVIEW` developer — Added blueprint.add_macro for empty macro graph creation with tunnel input/output and multi-exit signatures, plus regression coverage for pure macros, multi-exit exec pins, and graphName-scoped follow-up node creation.
- `#3-verify-add-macro` `DONE` tester — Verified: `blueprint.add_macro` on `/Game/App/UI/Test/W_McpVerifyTemp_F_blueprint_add_macro` returned `success=true`, `graphName=VerifyMacro`, tunnel node IDs, saved the multi-exit bool signature, and `blueprint.graph.create_node` scoped to `VerifyMacro` returned node `K2Node_CallFunction_0`; temp asset deleted with `asset.delete` and `existsAfter=false`.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
