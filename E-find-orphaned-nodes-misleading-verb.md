---
id: E-find-orphaned-nodes-misleading-verb
title: "`blueprint.graph.find_orphaned_nodes` is a mutating verb; `delete: true` rejected, must use `dryRun: false`"
status: DONE
severity: Low
category: ergonomic
tags: [blueprint-graph, naming, find-orphaned-nodes, dry-run]
---

# `blueprint.graph.find_orphaned_nodes` is a mutating verb; `delete: true` rejected, must use `dryRun: false`

`blueprint.graph.find_orphaned_nodes` reads like a query (`find_*`) but is actually a read-or-delete switch gated by `dryRun`. The natural intuition for "delete the orphans" is `delete: true`, which the handler rejects:

```
UNKNOWN_PARAMS: Unknown parameter(s) for 'blueprint.graph.find_orphaned_nodes': [delete]. Valid parameters: [assetPath, graphName, includeDataOnly, dryRun]
```

Handler `RPC_PARAMS` (`BlueprintGraphOrphanHandler.cpp:132-137`) only declares `assetPath`, `graphName`, `includeDataOnly`, `dryRun`. There is no `delete` alias. Same shape as the precedent `E-widget-bind-event-misleading-name`: action name and behavior don't match, agents pay a wrong-call tax to discover the convention.

The deeper bug: a single `find_*` RPC overloads query and mutation behind a double-negative flag (`dryRun: false` = "actually do it"). The `E-orphan-delta-cleanup` workaround in this same board even reaches for the mutating mode of this RPC, which reinforces that callers want the mutate path to be explicit.

**Workaround:** Pass `dryRun: false` to make it delete. Remember this is the opposite of intuition.

**Fix options (one of):**
1. **Split verbs (preferred):** keep `blueprint.graph.find_orphaned_nodes` as read-only (drop `dryRun`), add `blueprint.graph.delete_orphaned_nodes` for the mutating path. Matches the rest of the namespace (`delete_node`, `remove_event`).
2. **Alias:** accept `delete: true` as an alias for `dryRun: false` (and reject combinations that contradict). Smallest diff, but keeps the misleading verb.
3. **Rename in place:** rename to `blueprint.graph.sweep_orphaned_nodes` (or similar action verb) so the name signals mutation when `dryRun: false`.

## History
- `#1-delete-flag-rejected` `OPEN` reporter — Tried `blueprint.graph.find_orphaned_nodes(delete: true)`, got `UNKNOWN_PARAMS` with valid list `[assetPath, graphName, includeDataOnly, dryRun]`. Confirmed in `BlueprintGraphOrphanHandler.cpp:132-137`. Verb `find_*` reads as read-only but mutates when `dryRun: false`; this is the same misleading-name pattern as `E-widget-bind-event-misleading-name`. Cost: one wasted round-trip per agent that meets this RPC for the first time.
- `#2-split-find-delete-orphans` `IN-REVIEW` developer — Split blueprint.graph.find_orphaned_nodes into read-only find_orphaned_nodes (no dryRun, no deletedCount in response) and new mutating blueprint.graph.delete_orphaned_nodes in BlueprintGraphOrphanHandler.cpp. Extracted DeleteOrphans static helper wrapping FScopedTransaction + per-graph Modify + RemoveNode loop + MarkBlueprintAsStructurallyModified + CompileBlueprint + SaveLoadedAssetThrottled. Updated TestBlueprintGraphOrphan.cpp delete-mode tests to target the new RPC; added FDeleteOrphanedNodesHandlerTest covering delete then re-find clears. Dropped dryRun from any remaining test payloads. Updated docs/wiki/blueprint.graph.md.
- `#3-verify-split-schemas` `DONE` tester — Verified: `blueprint.graph.find_orphaned_nodes?` schema lists only `[assetPath, graphName, includeDataOnly]` (no `dryRun`); summary says "(read-only)". `blueprint.graph.delete_orphaned_nodes?` schema exists with same param shape and summary "Delete orphaned nodes... from a blueprint graph." Split verb fix is in place.
