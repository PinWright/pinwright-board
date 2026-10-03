---
id: E-compile-bpir-upsert-leaves-data-only-orphans
title: "compile_bpir: pure feeders orphaned by the Phase 0-pre ComponentEvent/WidgetEvent re-emit are never swept (Phase 0b only covers Replace-mode Phase 0 graphs)"
status: OPEN
severity: Low
rice: [2, 1, 1, 2]
priority: 8
category: ergonomic
tags: [bpir, orphans, cleanup, upsert]
---

# compile_bpir leaves pure-feeder orphans from the Phase 0-pre bound-event re-emit

All paths below are in `Source/PinWright/Private/Compiler/BpirCompiler.cpp`.

Phase 0-pre (`:3100-3158`, all modes) deletes an existing `UK2Node_ComponentBoundEvent` that a
ComponentEvent/WidgetEvent block re-emits, together with its subgraph
(`CollectSubgraphNodes`, `:2944`: downstream exec nodes plus pure feeders whose only consumers
are inside the deleted set, `CollectOwnedPureDependencies` `:2868`). Every block's set is
collected before any node is deleted (`:3111-3148`, delete loop `:3150-3157`), so a feeder
shared by two bound-event bodies that the same compile re-emits is kept by both collections, each
seeing the other body as a live consumer. It is an orphan once both bodies are gone.

Phase 0b, the sweep that removes pure nodes with every output unlinked (`:3459-3513`), does not
see those graphs:
- it runs only inside the Replace-mode block (`bPhase0Ran`, `:3185-3186`), so a Default-mode
  compile that re-emits a bound event gets no sweep at all;
- in Replace mode it scans only `AffectedGraphs`, built from Phase 0's `NodesToDelete`
  (`:3421-3428`), never from `BoundEventNodesToDelete` (`:3111`).

Observed: a 5-entry upsert on `W_DroneSelect_EditDrone` succeeded but left a
`K2Node_VariableGet 'Get Drone'` from a replaced body; `find_orphaned_nodes` reported
`orphanedCount 1` and a manual `delete_node` was needed.

**Workaround:** after `compile_bpir`, run `blueprint.compile` (it reports `orphanedCount` and
`orphanedNodes`) or `blueprint.graph.find_orphaned_nodes`, then delete the residue by nodeId or
with `blueprint.graph.delete_orphaned_nodes`, and save.

**Fix:** snapshot orphans with `BlueprintHandlerUtils::SnapshotBlueprintOrphanGuids` before
Phase 0-pre and remove only the new ones with `CleanupNewBlueprintOrphans` after Phase 0/0b
(`Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.h:756`, `:760`; already
used by `BlueprintGraphCrudHandler.cpp:800` and `WidgetHierarchyHandler.cpp:236`), in every
mode. Add the deleted count to `OrphansRemoved` (`:3949`). This also stops Phase 0b from
deleting orphans that existed before the compile. The cheaper partial fix is to add the
`BoundEventNodesToDelete` graphs to `AffectedGraphs` and run Phase 0b outside the Replace gate.
Either way, keep the Phase 0 rollback contract (`bPhase0Ran`): the cleanup must not track or
restore swept nodes.

**Acceptance:** a test builds a widget Blueprint with two bound-event bodies that share one
variable getter, then `compile_bpir`s both events in one Default-mode call; afterwards
`find_orphaned_nodes {includeDataOnly:true}` finds no new orphan and `orphansRemoved` counts the
getter. Run the same in Replace mode. An orphan that existed before the compile is still present.

## History
- `#1-residual-data-feeder-orphan` `OPEN` reporter — 5-entry upsert on W_DroneSelect_EditDrone succeeded (42 nodes) but left `K2Node_VariableGet 'Get Drone'` from a replaced body; `find_orphaned_nodes` → `orphanedCount 1`, manual `delete_node` required (other pure feeders pre-deleted manually, so leak count understated). Root cause: Phase 0b (`BpirCompiler.cpp:2866`) is scoped to `AffectedGraphs` (Phase 0 `NodesToDelete` only, not Phase 0-pre `BoundEventNodesToDelete`) and uses an all-outputs-disconnected heuristic; the robust delta cleanup built in `E-orphan-delta-cleanup` was deferred for this path.
- `#2-rephrased` `OPEN` developer — Every BpirCompiler.cpp line cite was stale (Phase 0-pre now :3100-3158, CollectOwnedPureDependencies :2868, AffectedGraphs :3421, Phase 0b :3459, OrphansRemoved :3949). Corrected them and stated two facts the old text missed: Phase 0b runs only in Replace mode (:3185-3186), so a Default-mode bound-event re-emit gets no sweep; and a feeder shared by two re-emitted bound-event bodies survives because all Phase 0-pre collections run before the delete loop. Dropped the unsupported "pre-orphaned earlier in the session" case (Phase 0b does sweep those in affected graphs). Named the delta helpers (BlueprintHandlerUtils.h:756/:760), added the cheaper partial fix, an acceptance test, and the blueprint.compile orphanedCount route to the workaround. Severity stays Low.
