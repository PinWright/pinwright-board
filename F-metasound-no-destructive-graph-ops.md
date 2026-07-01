---
id: F-metasound-no-destructive-graph-ops
title: "No MetaSound destructive graph ops (delete node, disconnect edge)"
status: DONE
severity: High
category: feature
tags: [audio, metasound, authoring, no-text-ir, builder]
---

# No MetaSound destructive graph ops (delete node, disconnect edge)

The `audio.authoring.*` MetaSound surface is mutate-only-forward. Today it
exposes `create_metasound`, `add_metasound_node`, `add_metasound_input`,
`add_metasound_output`, `connect_metasound_nodes`, `set_metasound_default`,
and (read-side) `describe_metasound`. **There is no
`remove_metasound_node`, `delete_metasound_node`, or
`disconnect_metasound_nodes`.** Verified by calling
`audio.authoring.remove_metasound_node` / `delete_metasound_node` /
`disconnect_metasound_nodes` — all return `Not found` with did-you-mean
suggestions, and a grep of `AudioAuthoringHandler.cpp` for
`REGISTER_RPC_HANDLER("audio.authoring.*metasound*"` returns exactly the
seven handlers above.

**Why this is higher severity than for other graph domains.** MetaSound
has **no text IR** (no MSIR / MGIR / BPIR equivalent). Blueprint, AnimBP,
and Material domains each ship a bulk text-IR path that can fully
replace a graph in one call, so the imperative-only gaps are mostly
ergonomic. MetaSound has only the imperative surface — if it lacks
`remove_node`, then **any incorrect node placement strands the graph
forever**. The agent can't undo an `add_metasound_node` mistake and
can't recover from `connect_metasound_nodes` wiring errors short of
deleting the entire asset and starting over. This is the single biggest
gating issue for agent-driven MetaSound authoring.

**Concrete blocked workflows:**

1. Iterative graph construction — agent adds a Multiply, realizes it
   needs an Add, can't remove the Multiply.
2. Rewiring — connecting `from_node.OutA -> to_node.InA` then realizing
   it should be `OutB -> InA` requires the disconnect.
3. Inspect-mutate-fix loop — `describe_metasound` shows the graph state
   but the agent has no way to act on the diagnosis.
4. Cleanup after `connect_metasound_nodes` failure — when a connect
   fails mid-build, dangling nodes accumulate with no removal path.

**Implementation surface:** `FMetaSoundFrontendDocumentBuilder` already
exposes the inverse ops; the existing handlers use this builder for
every add/connect call:

- `Builder.RemoveNode(FGuid NodeID)` — delete a graph node by GUID
- `Builder.RemoveNamedEdges(TSet<FNamedEdge>)` — delete edges by
  (sourceNodeID, sourceOutputName, targetNodeID, targetInputName) keys
- `Builder.RemoveEdge(FMetasoundFrontendEdge)` — alternative edge-by-id removal

Proposed RPCs (thin builder wrappers, mirroring the existing add/connect
handlers in shape):

```
audio.authoring.remove_metasound_node {
    assetPath: string,
    nodeId: string,        // GUID from add_metasound_node / describe_metasound
    save?: bool
} -> { message, removedEdges: number }

audio.authoring.disconnect_metasound_nodes {
    assetPath: string,
    sourceNodeId: string,
    sourceOutputName: string,
    targetNodeId: string,
    targetInputName: string,
    save?: bool
} -> { message, edgesRemoved: number }
```

The handlers share boilerplate with `add_metasound_node` /
`connect_metasound_nodes` (load asset, IMetaSoundDocumentInterface
cast, FMetaSoundFrontendDocumentBuilder construction, McpSafeAssetSave,
FinishBuilding under MCP_HAS_METASOUND_FRONTEND_V2). Adding the
inverses is a localized patch in `AudioAuthoringHandler.cpp`.

## History
- `#1-no-destructive-ops` `OPEN` reporter — Verified via `audio.authoring.remove_metasound_node` / `delete_metasound_node` / `disconnect_metasound_nodes` — all return Not found. Source grep on `REGISTER_RPC_HANDLER("audio.authoring.*metasound*"` returns only 7 handlers (create/add_node/connect/add_input/add_output/set_default/describe). MetaSound is uniquely affected because there's no text IR (no MSIR/MGIR/BPIR equivalent) — the imperative surface is the only authoring path, so missing inverses mean mistakes are permanent. Proposes thin wrappers around `FMetaSoundFrontendDocumentBuilder::RemoveNode` and `RemoveNamedEdges`, already used by the existing add/connect handlers.
- `#2-reviewed-and-confirmed` `OPEN` tester — Re-verified handler enumeration in `AudioAuthoringHandler.cpp`: lines 617/701/807/903/993/1083/2312 register exactly the 7 forward-mutate handlers; no `remove_*` or `disconnect_*` exist. No duplicate tickets in `docs/board/` (companion files cover other MetaSound gaps: input-output-mutation, interface-attach, patch-or-preset, variables-or-validate, describe, search-api, and the asset-dump no-graph-builder bug). Severity High remains justified — `asset.delete` workaround forces full restart (loses all prior graph state) and `property.set` to nullptr does not apply since MetaSound nodes are GUID-keyed inside `FMetaSoundFrontendDocumentBuilder`, not exposed as direct UObject properties. The crash ticket `B-metasound-add-input-asserts-on-missing-literal` is orthogonal: it gates `add_metasound_input` specifically, but `add_metasound_node` / `connect_metasound_nodes` mistakes are still unrecoverable once that crash is fixed, so this ticket's framing stands.
- `#3-add-remove-disconnect` `IN-REVIEW` developer — Added `audio.authoring.remove_metasound_node` and `disconnect_metasound_nodes` in new `Private/Handlers/Audio/MetaSound/MetaSoundDestructiveHandler.cpp`. Wrap `FMetaSoundFrontendDocumentBuilder::RemoveNode(FGuid)` and `RemoveNamedEdges(TSet<FNamedEdge>)` respectively, mirroring the load+cast+builder+FinishBuilding pattern from existing `add_metasound_node` / `connect_metasound_nodes`. Regression test `TestMetaSoundDestructiveOps.cpp` asserts engine builder API contracts the handlers depend on.
- `#4-skip-forward-add-blocked` `SKIP` tester — Cannot exercise the destructive ops through MCP because setup cannot create a removable MetaSound node: `audio.authoring.create_metasound` created `/Game/Audio/MetaSounds/MS_McpVerify_F_NoDestructive_20260515`, but `audio.authoring.add_metasound_node` returned `NODE_CLASS_NOT_FOUND` for shorthand `add`/`sine` and for `UE.Add.Float` returned by `audio.authoring.search_metasound_nodes`. The namespace index does list `remove_metasound_node` and `disconnect_metasound_nodes`, but live remove/disconnect behavior needs a valid node or edge.
- `#5-returned-disconnect-false-positive` `OPEN` tester — Returned: `audio.authoring.remove_metasound_node?` and `audio.authoring.disconnect_metasound_nodes?` are registered with the expected schemas, and `remove_metasound_node` on `/Game/Audio/MetaSounds/MS_PlayOneShot_2ch.MS_PlayOneShot_2ch` with nodeId `00000000-0000-0000-0000-000000000000` correctly returned `NODE_NOT_FOUND`; however, `disconnect_metasound_nodes` on the same asset with fake source/target GUIDs and fake pin names returned success with `edgesRemoved: 1` instead of reporting no matching edge. Test: `audio.authoring.disconnect_metasound_nodes` args `{"assetPath":"/Game/Audio/MetaSounds/MS_PlayOneShot_2ch.MS_PlayOneShot_2ch","sourceNodeId":"00000000-0000-0000-0000-000000000000","sourceOutputName":"Out","targetNodeId":"11111111-1111-1111-1111-111111111111","targetInputName":"In","save":false}`.
- `#6-reject-missing-disconnect-edge` `IN-REVIEW` developer — Fixed audio.authoring.disconnect_metasound_nodes in MetaSoundDestructiveHandler.cpp to use RemoveNamedEdges' removed-edge out parameter and report EDGE_NOT_FOUND when the requested edge tuple removes zero edges, including fake node IDs or pin names. Added regression test FDisconnectMetaSoundNodesRejectsMissingEdgeTest in TestMetaSoundDestructiveOps.cpp to exercise the production RPC handler with fake GUIDs and assert it fails instead of returning edgesRemoved: 1.
- `#7-verify-edge-not-found` `DONE` tester — Verified: `audio.authoring.disconnect_metasound_nodes` on `/Game/Audio/MetaSounds/MS_PlayOneShot_2ch.MS_PlayOneShot_2ch` with fake GUIDs `00000000-...` / `11111111-...` and pin names `Out`/`In` now returns `EDGE_NOT_FOUND` with message naming both node IDs and pin names, instead of the previous false-positive `edgesRemoved: 1`. Sanity-checked `audio.authoring.remove_metasound_node` on the same asset with fake nodeId — still returns `NODE_NOT_FOUND` as expected from `#5`.
