---
id: E-orphan-delta-cleanup
title: "Delete newly orphaned graph nodes by default"
status: DONE
severity: Medium
category: ergonomic
tags: [blueprint-graph, cleanup, deletion]
---

# Delete Newly Orphaned Graph Nodes by Default

Graph deletion RPCs currently remove the requested event, node, widget-bound
event, or exec-owned chain, but data-only nodes that become unreachable can be
left behind as orphaned graph fragments. This is especially easy to miss when
the reachable behavior decompiles cleanly while old pure nodes remain parked in
the graph.

Deletion helpers should share a common orphan-delta cleanup path:

1. Snapshot the affected graph or asset orphan set before the mutation.
2. Perform the requested deletion.
3. Snapshot the orphan set again.
4. Compute `newOrphans = after - before`.
5. Delete `newOrphans` by default.
6. Report every newly orphaned node that was deleted.

This must be delta-based, not dependency-based. Do not delete arbitrary data
dependencies of the removed entry, because pure/data nodes can be shared by
other reachable events, functions, or macros. If a node is still reachable, it
will not appear in the post-mutation orphan delta and must not be deleted.

Apply the shared cleanup to deletion surfaces such as `blueprint.remove_event`,
`blueprint.graph.delete_node`, BPIR replace/block deletion paths, and
`widget.remove_widget` when it removes bound events or widget variables.

**Workaround:** After graph deletion, manually run
`blueprint.graph.find_orphaned_nodes(dryRun=false, includeDataOnly=true)` on the
affected asset, then compile and save.

**Fix:** Add a shared Blueprint orphan-delta cleanup utility that snapshots reachable-orphan state before a mutation, computes newly orphaned nodes after the mutation, deletes only that delta by default, and reports the affected nodes. In this sprint, apply it to direct deletion surfaces: `blueprint.graph.delete_node`, `blueprint.remove_event`, and `widget.remove_widget` after its bound-event cascade. Add `cleanupNewOrphans=false` as a report-only escape hatch. Leave BPIR replace/upsert cleanup as a follow-up unless it can reuse the helper without changing compile-result ownership or rollback semantics.

## History
- `#1-initial-request` `OPEN` reporter — Deletion RPCs should delete nodes that become orphaned by the operation by default and report what was removed, using an orphan-delta approach so shared reachable data nodes are preserved.
- `#2-orphan-delta-cleanup` `IN-REVIEW` developer — Added shared Blueprint orphan-delta cleanup, applied it to direct deletion RPCs, and covered `blueprint.graph.delete_node` with `FDeleteNodeCleansNewOrphanDeltaTest`.
- `#3-verify-delete-node-cleanup` `DONE` tester — Verified: created `/Game/App/UI/Test/W_McpVerifyTemp_E_orphan_delta_cleanup`, compiled a BeginPlay BPIR graph with `Conv_IntToString` feeding `PrintString`, then deleted the `PrintString` node via `blueprint.graph.delete_node`; response reported `cleanupNewOrphans: true`, `newOrphanedCount: 1`, and `deletedNewOrphanCount: 1` for the pure `To String (Integer)` node, and a follow-up `blueprint.graph.find_orphaned_nodes` returned `orphanedCount: 0`. Temp asset was deleted.
