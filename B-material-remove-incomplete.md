---
id: B-material-remove-incomplete
title: "material.graph.remove_node removes only the collection entry and leaves live wires and parameter bookkeeping behind"
status: IN-REVIEW
severity: High
category: bug
tags: [material, graph, remove-node, dangling-connection, bookkeeping, false-success]
---

# The response says removed while the expression can remain in use

`material.graph.remove_node` calls `FMaterialMutationTarget::RemoveExpression`, then
`NotifyEdited`, and returns `{removed:true}` (`MaterialGraphHandler.cpp:98-135`). The shared helper
only removes the pointer from `ExpressionCollection.Expressions`
(`MaterialFinders.h:193-201`). It does not break inputs in other expressions, clear a main-material
input, remove parameter metadata, or mark the expression as garbage.

UE 5.8's exported deletion helpers show the required behavior. `DeleteMaterialExpression` first
breaks every link to the expression, clears every material-property input, calls
`RemoveExpressionParameter`, removes the collection entry, and marks the expression as garbage
(`MaterialEditingLibrary.cpp:570-616`). The function variant likewise breaks links and marks the
object as garbage (`:1353-1363`).

Consequently, deleting a node that feeds another node or a root output can leave that live pointer
in the material while discovery no longer lists its source. Parameter caches can also retain an
entry for a node reported removed. This is a silent incomplete mutation, not merely missing Undo.

## What it should do

Route material and material-function deletion through the corresponding
`UMaterialEditingLibrary` helper, or mirror all of its cleanup legs in `FMaterialMutationTarget`.
The regression should delete a wired parameter node and prove the collection, downstream input,
root input, parameter metadata, and garbage state all changed before success.

## Workaround

Break all inbound/root connections before removal and re-open or decompile the material to verify
the graph rather than trusting `removed:true`.

## Related

- `B-material-graph-no-transaction`

## Fix

The mutation helper previously removed only the expression-collection pointer and reported success, leaving surviving wires, material root inputs, parameter bookkeeping, and a live object behind. It now validates ownership and collection membership, delegates material and function deletion to UE's exported `UMaterialEditingLibrary` helpers, and returns success only after the expression is gone and marked with the internal `EInternalObjectFlags::Garbage` state.

Files changed:
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Material\MaterialFinders.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Material\MaterialGraphHandler.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Material\TestMaterialGraphRemoveNode.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Material\MaterialTestHelpers.h` (shared fixture/count helpers)
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Docs\wiki-src\material.graph.md`

Exact tests:
- `PinWright.material.graph.remove_node.CleansUpMaterialExpression`
- `PinWright.material.graph.remove_node.CleansUpMaterialFunctionExpression`

Deliberate non-changes: the native helper's contract does not promise clearing the deleted node's own inputs or execution begin/end links; `removed:true` means graph deletion, not shader compilation. No copied engine cleanup, engine-source edits, builds, live PIE, or commits were made.

## History
- `#1-source-scan` `OPEN` reporter -- Compared current PinWright deletion with UE 5.8's exported
  material and material-function deletion implementations; source-only under scan constraints.
- `#2-native-delete-path` `IN-REVIEW` developer -- Routed remove_node through native material/function cleanup with focused RPC regression coverage; static verification only.
