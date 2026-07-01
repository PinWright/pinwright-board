---
id: B-material-get-node-details-list-mode-errors
title: "material.graph.get_node_details list-all mode returns NODE_NOT_FOUND error instead of the node list"
status: IN-REVIEW
severity: High
category: bug
tags: [material, material-graph, get-node-details, list-mode, error-envelope, documented-input-rejected]
---

# `material.graph.get_node_details` with no `nodeId` is wrongly returned as an error

The wiki and the handler registration both document a list-all mode:
the summary says *"...or list every node in the graph when nodeId is
omitted."* and the param doc says *"`nodeId` (`string`, optional): Node
GUID/name (omit to list all nodes)"*. Omitting `nodeId` is therefore a
**valid documented input**.

The handler actually **builds** the full node list, then throws it away by
returning it through `SendError` with code `NODE_NOT_FOUND` instead of
`SendSuccess`. In `MaterialGraphHandler.cpp` (the `else` branch of
`material.graph.get_node_details`, ~L341-366):

```cpp
else
{
    TSharedPtr<FJsonObject> Result = MakeShared<FJsonObject>();
    TArray<TSharedPtr<FJsonValue>> NodeList;
    for (int32 i = 0; i < AllExpressions.Num(); ++i) { /* ...build NodeInfo... */ }
    Result->SetArrayField(TEXT("availableNodes"), NodeList);
    Result->SetNumberField(TEXT("nodeCount"), AllExpressions.Num());

    FString Message = NodeId.IsEmpty()
        ? FString::Printf(TEXT("No nodeId provided. Material has %d nodes."), AllExpressions.Num())
        : FString::Printf(TEXT("Node '%s' not found. Material has %d nodes."), *NodeId, AllExpressions.Num());

    Ctx.SendError(TEXT("NODE_NOT_FOUND"), Message);   // <-- list-all path lands here too
}
```

This branch is shared between two distinct cases: (a) a `nodeId` was
supplied but did not resolve (a genuine error), and (b) **no `nodeId` was
supplied at all** (the documented list-all mode, which should succeed).
Both are funneled into `SendError(NODE_NOT_FOUND)`, so the documented
list-all input is wrongly rejected. The freshly-built `availableNodes` /
`nodeCount` `Result` object is constructed and then discarded — it is never
sent through `SendSuccess`, so it does not reach the caller as
`structuredContent`; the call surfaces as `isError: true`.

**Why it matters:**

- The only graph-wide enumeration entry point on this method is
  unreachable as a success. A caller that follows the documentation
  (`get_node_details` with just `assetPath` to dump the graph) gets an
  error envelope and must fall back to `material.authoring.get_material_info`
  for node enumeration — exactly the workaround the attempt agent took.
- It is the natural first call when reading back a graph after authoring
  (add_expression → connect_nodes → "now show me the graph"). Returning an
  error here makes round-trip authoring feel broken even though the data is
  right there in the discarded `Result`.
- Sibling note: `B-material-get-node-details-missing-pins-props` (DONE)
  deliberately left "Bulk-list branch unchanged to keep responses compact"
  — that ticket was about *missing fields* in the payload and did not touch
  the error-vs-success envelope, so this is not a duplicate.

**Fix:**

Split the shared `else` branch on `NodeId.IsEmpty()`. When `NodeId` is
empty (list-all mode), send the already-built `Result`
(`{availableNodes, nodeCount}`) via `Ctx.SendSuccess(Result)`. Only the
"nodeId supplied but not found" case should remain `SendError(NODE_NOT_FOUND)`.

## Repro

1. `material.authoring.create_material(path: "/Game/Materials/M_EnergyShield",
   domain: Surface, blendMode: Translucent)` (any material with >=1 node
   works).
2. `material.graph.get_node_details(assetPath: "/Game/Materials/M_EnergyShield")`
   — **no `nodeId`**.
3. Observed: `isError: true`, error text
   `[NODE_NOT_FOUND] No nodeId provided. Material has 6 nodes.`
   (replay-confirmed live on this build against the existing
   `/Game/Materials/M_EnergyShield`).
4. Expected per wiki: a success result listing every node
   (`{availableNodes: [...], nodeCount: 6}`).

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live: `material.graph.get_node_details(assetPath: /Game/Materials/M_EnergyShield)` with no `nodeId` returns `[NODE_NOT_FOUND] No nodeId provided. Material has 6 nodes.` (`isError: true`). The handler's `else` branch builds the `availableNodes`/`nodeCount` `Result` and then discards it via `SendError` instead of `SendSuccess`, so the documented "omit to list all nodes" input is wrongly rejected. Root cause: the empty-`nodeId` list-all case shares the same `else` branch as the genuine "nodeId not found" error. Fix: branch on `NodeId.IsEmpty()` and send the built `Result` via `SendSuccess` for the list-all case.
- `#2-fix-implemented` `IN-REVIEW` developer — Split the shared `else` branch on `NodeId.IsEmpty()` in `material.graph.get_node_details`: the list-all case (no `nodeId`) now sends the already-built `{availableNodes, nodeCount}` `Result` via `Ctx.SendSuccess` (with `AddAssetVerification`, mirroring the single-node success branch); only the genuine "nodeId supplied but not found" case keeps `SendError(NODE_NOT_FOUND)`. The discarded-payload bug is gone — the documented list-all input is now a success envelope. File: `Source/EditorAutomationRpcGateway/Private/Handlers/Material/MaterialGraphHandler.cpp` (~L341-371). Regression test: added `FMaterialGetNodeDetailsListAllTest` (`EditorAutomationRpcGateway.material.graph.get_node_details.ListAllMode`) in `Source/EditorAutomationRpcGateway/Private/Tests/Material/TestMaterialGetNodeDetails.cpp` — creates a one-node material, invokes the handler with no `nodeId`, and asserts `Capture.bSuccess`, empty `ErrorCode`, and a non-empty `availableNodes` array + `nodeCount >= 1` in the structured result; it would fail (on `bSuccess`/`ErrorCode`) if the fix were reverted. Not compiled/run here — a later phase drives compile+tests.
