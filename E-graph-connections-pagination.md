---
id: E-graph-connections-pagination
title: "`blueprint_graph_get_graph_connections` has no size limits or filters"
status: DONE
severity: Low
category: ergonomic
tags: []
---

# `blueprint_graph_get_graph_connections` has no size limits or filters

`blueprint_graph_get_graph_connections` returns every edge in a graph in a single payload. On mid-sized widgets like `W_LyraFrontEnd` (74-node EventGraph), the response was 66,260 chars / 2,278 lines, overflowing the MCP client context and forcing a fallback to reading the saved tool-result file via grep.

**Workaround:**
Read the saved tool-result file directly via filesystem grep scoped to a known nodeId.

**Proposal:**
Add optional parameters:
- `nodeId` / `nodeIds` (array) — return only edges where `fromNodeId` or `toNodeId` is in the given set (primary use case: tracing outputs from a known event node).
- `edgeType` — filter to `exec` or `data`.
- Or return a structured summary + pagination cursor for extremely large graphs.

## History
- `#1-full-graph-dump-overflow` `OPEN` reporter — Hit while tracing the exec flow from `On Clicked (W_ConstructButton)` in `W_LyraFrontEnd` during Task 0 probe of replay subtask #03. Full-graph dump exceeded the 30k-char context limit; had to grep the saved tool-result file by `fromNodeId` to get the downstream node.
- `#2-added-filter-params` `IN-REVIEW` developer — Added `nodeIds`, `edgeType`, `maxEdges` optional params to `blueprint.graph.get_graph_connections`. Filtering applied inside `BuildGraphConnectionsJson` iteration (skip non-matching edges before push; track `totalMatched` separately from capped output). Response gained `truncated` and `totalMatched` fields. Existing fields unchanged for backward compat. `nodeIds` parsed via `Ctx.GetArray` into a `TSet<FString>` using the pattern from `get_node_details_batch`.
- `#3-verified-filter-params-work` `DONE` tester — Verified on `/App/App/UI/LobbyAndMenu/W_LyraFrontEnd` EventGraph (324 edges: 145 exec + 179 data). Schema exposes `nodeIds`/`edgeType`/`maxEdges`. Baseline: `connectionCount:324, truncated:false, totalMatched:324`. `nodeIds:[one-id]` returned 9 edges all touching that id, `totalMatched:9`. `edgeType:"exec" + maxEdges:5` returned 5 exec edges with `truncated:true, totalMatched:145` (un-capped total reported correctly). `edgeType:"data" + maxEdges:5` returned 5 data edges, `totalMatched:179`. Backward-compat fields preserved. All new params work.
- `#4-residual-full-guid-verbosity` `DONE` reporter — Cross-task evidence (struggle audit, focus `blueprint.compile_bpir`, `/Game/BP_RoundTripProbe`, 24-node graph). NOT reopening — the filter fix here works and is the right mitigation. Recording a residual: even *with* `edgeType:"exec"`, the exec-only dump came back at 10236 chars — barely (236) over the 10000 spill threshold — purely because each edge repeats two full 32-char node GUIDs, so it spilled to `HttpResponses\…json` and forced a `Read`+parse to inspect wiring (the goal was ALL exec edges, so capping with `maxEdges` would have defeated the verification). Possible future enhancement if revisited: a compact/short-id edge form or `from→to` label summary so a full exec-wiring read of a moderate graph stays inline. Low value given the spill+Read fallback (`E-http-response-spill`, DONE) is the designed path and `nodeIds`/`maxEdges` already exist; logged for aggregation only.
