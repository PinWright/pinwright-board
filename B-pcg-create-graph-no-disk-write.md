---
id: B-pcg-create-graph-no-disk-write
title: "pcg.create_graph reports success for a graph that exists only in memory because McpSafeAssetSave never writes the package to disk"
status: OPEN
severity: High
category: bug
tags: [pcg, create-graph, persistence, no-disk-write, false-success, data-loss, mcp-safe-asset-save]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# `pcg.create_graph` returns a usable object path without making the graph durable

> **SOURCE-ONLY.** This scan did not run the editor or call the verb.

`PCGGraphCreate.cpp:104-125` creates the package and graph, publishes it to the Asset Registry,
then calls `McpSafeAssetSave(Graph)`. That helper is the project's mark-dirty compatibility seam;
it does not perform a package write. The response at `:127-134` contains `graphPath`, `name`,
`graphClass`, and `AddAssetVerification`, then returns success with no `saveRequested`, `saved`,
`saveState`, `pendingFlush`, file size, or disk re-load proof.

A caller can add nodes to the returned in-memory graph and treat it as a created asset, but a cold
editor launch has no `.uasset` to load unless the caller separately discovers and invokes
`asset.save`. This is the catalog's `memory-state-masquerades-as-durable-save` and
`persistence-without-disk-proof` shape, not an Asset Registry failure.

## What should happen

Use `SaveAssetToDiskReportingPresence` and emit the shared `AddAssetSaveReport` contract, including
the actual `saveState` and disk size. If creation is intentionally dirty-only, say that explicitly
and do not present the Asset Registry presence check as persistence. A regression must cold-load or
otherwise prove a file exists after `pcg.create_graph` returns.

**Workaround:** immediately call `asset.save {assetPath:<graphPath>, force:true}` and require its
durability result before authoring against the graph.

## Related

`B-geometry-generate-lods-no-disk-write` is the same save-helper mistake in the LOD family; its
accepted fix is the reusable save/report shape. No existing PCG ticket covers graph durability.

## History
- `#1-source-pattern-scan` `OPEN` reporter — The graph creation path ends in `McpSafeAssetSave` and an in-memory verification block, with no disk write or persistence fields. Source-only; no editor, build, or test was run. Fix with the shared measured save path and require disk proof.
