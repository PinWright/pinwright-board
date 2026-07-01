---
id: E-get-nodes-no-count-field
title: "`blueprint.graph.get_nodes` returns a bare nodes array with no top-level nodeCount"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [blueprint, response-shape]
encounters: 1
lastSeen: 2026-06-24T06:10:13Z
---

# `blueprint.graph.get_nodes` returns a bare nodes array with no top-level nodeCount

`blueprint.graph.get_nodes` returns `{ nodes: [...], graphName, verification }` with **no top-level count field**. To answer "how many nodes are in this graph?" the caller must either length-count the `nodes` array client-side or, when the response has spilled to an on-disk `HttpResponses` JSON (the common case for a large EventGraph), grep the saved file for top-level node entries. Every sibling read-only method in the same `blueprint.graph.*` inspection family already returns a count summary, so `get_nodes` is the lone outlier:

- `get_graph_details` → `nodeCount` (`BlueprintGraphInspectionHandler.cpp:487` — `Result->SetNumberField(TEXT("nodeCount"), TargetGraph->Nodes.Num())`)
- `get_graph_connections` → `connectionCount` + `totalMatched` (cpp:569, 571)
- `get_execution_flow` → `nodeCount` / `totalNodeCount` (cpp:1528–1582)
- `list_node_types` → `count` (cpp:1198)
- `get_nodes` → **only `nodes`, `graphName`** (cpp:431–436) — no count

This is purely a response-shape consistency gap: the same numeric summary that callers rely on everywhere else in this family is missing from the one method whose entire purpose is "list all nodes."

**Workaround:** length-count the returned `nodes` array, or — when the response spilled to disk — `Read`/`Grep` the saved tool-result file and count top-level node entries.

**Fix:** add `Result->SetNumberField(TEXT("nodeCount"), NodesArray.Num())` to the `get_nodes` handler in `BlueprintGraphInspectionHandler.cpp` (the block around cpp:431, right next to the existing `SetArrayField(TEXT("nodes"), ...)`), mirroring `get_graph_details`. One-line change; existing fields unchanged for backward compat.

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of a clean BP_Gears walkthrough task (focus `blueprint.graph.get_graph_details`, 13 calls, all ok). The sole reported friction, quoted verbatim: *"only minor friction was get_nodes having no top-level nodeCount field, so I confirmed the count by grepping top-level node entries (20)."* Confirmed in source: `get_nodes` (BlueprintGraphInspectionHandler.cpp:431–436) returns `nodes`+`graphName` only, while siblings `get_graph_details` (cpp:487 `nodeCount`), `get_graph_connections` (cpp:569 `connectionCount`), `get_execution_flow` (cpp:1528–1582 `nodeCount`/`totalNodeCount`), and `list_node_types` (cpp:1198 `count`) all expose a count. Lone outlier in the inspection family; one-line fix.
- `#2-already-fixed` `IN-REVIEW` developer — Already resolved in current source; no code change made. The requested `Result->SetNumberField(TEXT("nodeCount"), NodesArray.Num())` already exists verbatim at `Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphInspectionHandler.cpp:475`, immediately after `SetArrayField(TEXT("nodes"))` (:468); adjacent lines also emit `totalNodeCount` (:476) and `truncated` (:477), and the :470-474 comment notes it mirrors the get_execution_flow count triplet to keep the blueprint.graph family uniform. `git blame`/`git log -L 475,477` show the triplet landed in committed HEAD commit 9aed16c (the sibling `E-get-nodes-pins-spill-no-projection` projection/limit fix, which touched this same Result block), with a clean working tree. The ticket's cited line range (cpp:431-436) is stale — that range is now per-node field-setting, not the Result shape. Regression coverage already exists: `FBlueprintGraphGetNodesProjectionAndLimitTest` (Tests/Blueprint/TestBlueprintHandlers.cpp) asserts `totalNodeCount`/`truncated`. Flipped OPEN -> IN-REVIEW for a tester to verify the field is present; released the fuzz3 claim.
