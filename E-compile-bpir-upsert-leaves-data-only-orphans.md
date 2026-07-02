---
id: E-compile-bpir-upsert-leaves-data-only-orphans
title: "compile_bpir upsert leaves residual data-feeder orphans Phase 0b's graph-scoped sweep misses"
status: OPEN
severity: Low
category: ergonomic
tags: [bpir, orphans, cleanup, upsert]
---

# compile_bpir upsert leaves residual data-feeder orphans Phase 0b's graph-scoped sweep misses

The BPIR upsert (replace-mode) deletion already pulls owned pure feeders into the swept set: `CollectSubgraphNodes` runs `CollectDownstreamExecNodes` then `CollectOwnedPureDependencies` (`BpirCompiler.cpp:2340-2427`), which adds upstream pure nodes whose only consumers are inside the deletion set (shared feeders are deliberately kept). Phase 0b (`BpirCompiler.cpp:2866-2919`) then fixed-point-deletes any remaining all-outputs-disconnected pure node — but only within `AffectedGraphs`, which is built solely from Phase 0's `NodesToDelete` (`:2827-2835`), NOT from the separate Phase 0-pre `BoundEventNodesToDelete` pass (`:2554-2600`). So a pure feeder orphaned by a Phase 0-pre bound-event deletion, or one shared across two bodies swept in separate passes, or one pre-orphaned earlier in the session, can survive both the owned-dependency collection and Phase 0b. This session, a 5-entry upsert on W_DroneSelect_EditDrone left a `K2Node_VariableGet 'Get Drone'` from a replaced body that `find_orphaned_nodes` flagged (orphanedCount 1); a manual `delete_node` was required.

This is NOT the "exec-only sweep" it first appears to be (owned pure feeders are already collected), and NOT a regression of `E-replace-auto-clean` (its Phase 0b common case still works — verified there). It is the follow-up `E-orphan-delta-cleanup` (DONE) explicitly deferred: "Leave BPIR replace/upsert cleanup as a follow-up unless it can reuse the helper without changing compile-result ownership or rollback semantics."

**Workaround:** After an upsert, run `blueprint.graph.find_orphaned_nodes(includeDataOnly=true)` (now the default) and `delete_node` any residue, then recompile/save.
**Fix:** Replace the graph-scoped, all-outputs-disconnected Phase 0b heuristic with the delta-based orphan-cleanup helper from `E-orphan-delta-cleanup`: snapshot the reachable-orphan set before Phase 0-pre + Phase 0, snapshot after, delete only the new-orphan delta (so shared/reachable feeders stay), and fold the count into `OrphansRemoved`. Widen the snapshot window to cover the Phase 0-pre bound-event pass. Preserve the Phase 0 rollback contract (`bPhase0Ran`) — the helper must not restore or track swept nodes.

## History
- `#1-residual-data-feeder-orphan` `OPEN` reporter — 5-entry upsert on W_DroneSelect_EditDrone succeeded (42 nodes) but left `K2Node_VariableGet 'Get Drone'` from a replaced body; `find_orphaned_nodes` → `orphanedCount 1`, manual `delete_node` required (other pure feeders pre-deleted manually, so leak count understated). Root cause: Phase 0b (`BpirCompiler.cpp:2866`) is scoped to `AffectedGraphs` (Phase 0 `NodesToDelete` only, not Phase 0-pre `BoundEventNodesToDelete`) and uses an all-outputs-disconnected heuristic; the robust delta cleanup built in `E-orphan-delta-cleanup` was deferred for this path.
