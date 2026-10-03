---
id: B-material-graph-no-transaction
title: "All material.graph mutators run without an FScopedTransaction, so editor.undo cannot restore their changes"
status: DONE
severity: Medium
category: bug
tags: [material, graph, undo, transaction, authoring]
---

# Seven material graph write verbs have no transaction

`MaterialGraphHandler.cpp` registers seven mutators: `add_node`, `remove_node`, `connect_nodes`,
`break_connections`, `add_texture_sample`, `add_expression`, and `create_nodes`. The file contains
no `FScopedTransaction`. Three later creation paths call `Modify()`, while the first four rely on
direct collection/input changes and `PostEditChange`; none has an active transaction in which an
undo record could be stored.

The result is most costly for `remove_node`: losing a configured expression cannot be recovered by
Undo. Connections and batch-created nodes likewise have to be found and repaired manually. UE's
Material Editor deletion path opens an `FScopedTransaction` and modifies/breaks nodes before
removal (`MaterialEditor.cpp:6245-6249`, `:6340-6402`), providing the local engine pattern.

## What it should do

Open one scoped transaction per RPC and call `Modify()` on the material/function plus objects whose
serialized fields change. A batch create should remain one undo step. Add Undo/Redo coverage for a
wire, a created node, and a configured node deletion.

## Workaround

Save/revision-control the asset before mutation and revert the package when a graph edit must be
undone.

## Related

- `B-material-remove-incomplete`
- `B-pcg-graph-mutators-no-transaction`

## History
- `#1-source-scan` `OPEN` reporter -- Source-only census of all registrations and transaction/
  `Modify` calls in the handler; no Undo execution under the scan constraints.
- `#2-one-scoped-transaction-per-mutator` `IN-REVIEW` developer — Still reproducible: `MaterialGraphHandler.cpp` had zero `FScopedTransaction`. Fix is in one shared place, `PinWright::Material::FScopedMaterialGraphEdit` (`Handlers/Material/MaterialFinders.h`): opens `FScopedTransaction` titled `PinWright: <method>` and `Modify()`s the graph owner (UMaterial::Modify forwards to UMaterialEditorOnlyData, which holds the expression collection and main inputs; UMaterialFunction::Modify does NOT forward, so the function's editor-only data is Modify()d explicitly); `Cancel()` drops the step on a post-open refusal, and only when this scope is outermost (`UTransBuffer::Cancel` cannot cancel a nested part and would drop the caller's outer transaction). All seven mutators construct exactly one after validation: `add_node`, `remove_node`, `connect_nodes` (+ `TargetExpr->Modify()` for an expression pin), `break_connections` (+ `TargetExpr->Modify()`), `add_texture_sample`, `add_expression`, `create_nodes` (one step for the whole batch; cancelled when nothing was created) - replacing the three bare `Modify()` calls outside any transaction. `FMaterialMutationTarget::RemoveExpression` now calls `ModifyExpressionAndReferencers` before the native `UMaterialEditingLibrary` delete, so the victim (MarkAsGarbage, restored by the transactor's PendingKillChange) and every sibling whose input `BreakLinksToExpression` clears are recorded, mirroring the Material Editor's GraphNode->Modify() before BreakAllNodeLinks. `material.authoring.auto_layout` is untouched (it has its own transaction and does not route through this helper). Docs: `docs/wiki-src/material.graph.md` new `## Undo` section; CHANGELOG. Tests (`Tests/Material/TestMaterialGraphUndo.cpp`; each asserts the NEWEST undo entry is the verb's own titled transaction containing the edited object BEFORE calling `GEditor->UndoTransaction`, so a reverted fix fails there instead of undoing another test's step; fixtures are created with RF_Transactional like a factory-made asset): `PinWright.material.graph.undo.ConnectNodesUndoesAndRedoes` (expression pin undo+redo, main BaseColor undo), `PinWright.material.graph.undo.CreateNodesBatchIsOneUndoStep` (3-node batch removed by ONE undo, redo restores), `PinWright.material.graph.undo.RemoveNodeRestoresConfiguredNodeAndWires` (configured ScalarParameter name/value, sibling wire and BaseColor wire restored, victim valid again; transaction contains the sibling; redo removes again), `PinWright.material.graph.undo.EveryMutatorRecordsItsOwnStep` (add_node on a material AND a material function, add_expression, add_texture_sample, break_connections node + Main pin, each undone; a refused add_expression with an unknown property leaves the queue length unchanged).
- `#3-review-fixes-non-transactional-assets` `IN-REVIEW` developer — Independent review found a blocker in `#2`: the plugin's own creators (`material.authoring.create_material` and siblings, MGIR) create materials/functions WITHOUT `RF_Transactional`, so `Modify()` recorded nothing for the owner/editor-only data while the new expressions recorded their creation, and undo left a dead collection entry (add_node) or revived a node outside the collection (remove_node). Fixed with `PinWright::Material::RecordForUndo` (`MaterialFinders.h`): `SetFlags(RF_Transactional)` then `Modify(/*bAlwaysMarkDirty=*/false)`, as the Material Editor does; used for the owner AND its editor-only data (material and function), for `connect_nodes` / `break_connections` target expressions and in `ModifyExpressionAndReferencers`. `bAlwaysMarkDirty=false` means a refusal no longer dirties the package (every success path dirties it itself). `break_connections` now resolves the named pin before opening the transaction. `RemoveExpression` reports whether the native delete ran; `remove_node` only cancels when it did not, so a partial delete keeps its undo step. `create_nodes` ends its transaction (`TOptional::Reset`) before verification / `waitForShaderCompile`. Wiki `## Undo` rewritten (nested-transaction merge, refusal/dirty behaviour, RF_Transactional note, no history sentence); CHANGELOG notes the flag. New test `PinWright.material.graph.undo.NonTransactionalAssetUndoesCleanly` (fixture created with RF_Public|RF_Standalone only, asserted non-transactional; undo of add_node restores the count with no null/garbage entry; undo of remove_node puts the victim back IN the collection with its sibling wire). fastcheck OK on MaterialGraphHandler/MaterialDiscoveryHandler/MaterialAuthoringHandler/TestMaterialGraphUndo; check_test_ids / check_test_skips CLEAN.
- `#4-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit 664d0c68). run3/full passed non-skipped, all from `PinWright.material.graph.undo`: `.ConnectNodesUndoesAndRedoes`, `.CreateNodesBatchIsOneUndoStep`, `.RemoveNodeRestoresConfiguredNodeAndWires`, `.EveryMutatorRecordsItsOwnStep` and `.NonTransactionalAssetUndoesCleanly`. The ticket asked for undo/redo coverage of three cases, all met. A wire: connect_nodes on an expression pin and on main BaseColor. A created node: a 3-node create_nodes batch is one undo step, and redo restores it. A configured-node deletion: remove_node restores the ScalarParameter name/value, the sibling and BaseColor wires and collection membership, and redo removes it again. Each test first asserts that the newest undo entry is the verb's own titled transaction. All seven mutators record a step on a material and on a material function. A refused add_expression leaves the queue unchanged, and assets created without RF_Transactional undo cleanly.
