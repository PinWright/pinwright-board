---
id: E-execflow-no-event-entry-timeline-root
title: "get_execution_flow auto-start fails on a Timeline-rooted EventGraph (no event/function entry) with an unhelpful NODE_NOT_FOUND"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [blueprint, execution-flow, entry-points, k2node-timeline, auto-start, discoverability]
---

# get_execution_flow auto-start fails on a Timeline-rooted EventGraph with an unhelpful NODE_NOT_FOUND

`blueprint.graph.get_execution_flow` documents `startNodeId` as optional —
"defaults to first event/entry" — and offers two bulk discovery modes
(`includeAllEntryPoints`, `entryPointsOnly`). All three convenient modes fail
on a perfectly valid, idiomatic Blueprint whose EventGraph has **no
`K2Node_Event` / `K2Node_FunctionEntry` node** because execution is driven by an
**auto-play `K2Node_Timeline`**. The entry-point collector recognizes only
event/function-entry nodes as roots; it does not treat a latent
event-producing node (a Timeline, whose `Update`/`Finished`/`Impact` exec
**output** pins are the de-facto roots of the graph) as an entry point.

The defect is ergonomic, not a hard bug: the error is factually accurate (the
graph genuinely has no event/function entry node) and the tool can still trace
the graph — but only if the caller already knows to pass the Timeline node as
`startNodeId`. The contradiction is the awkward part: **`get_execution_flow`,
handed that same Timeline node as `startNodeId`, walks the full chain
correctly and reports `entryPoints: []`** — so the tool can root execution at
a Timeline, it just won't auto-discover it. The auto-start / bulk-discovery
modes leave the agent with a `NODE_NOT_FOUND` that names no remedy (no hint
that a `startNodeId` would work, no hint that latent roots exist), forcing a
detour through `get_graph_details` / `find_nodes` to find the Timeline's GUID
by hand. Auto-play timelines with no BeginPlay are a common shipped pattern —
this repro is an Epic Content Examples asset.

## Quotable demonstration (verbatim repro via mcp__editor-automation__call)

Asset: `/Game/ExampleContent/Blueprints/Blueprints/BP_Timeline_Ball.BP_Timeline_Ball`,
graph `EventGraph` (ubergraph, 10 nodes; the only execution root is
`K2Node_Timeline` "Bounce", nodeId `AFB695704811711CF90345BB38C4C6BA`, whose
own comment reads "This timeline is set to play and loop automatically"). The
graph contains zero `K2Node_Event` / `K2Node_FunctionEntry` nodes.

1. Default auto-start (no `startNodeId`):
   `get_execution_flow {assetPath:".../BP_Timeline_Ball", graphName:"EventGraph"}`
   → error: `[NODE_NOT_FOUND] Could not find a starting node (event or function entry).`

2. Bulk entry-point discovery:
   `get_execution_flow {... graphName:"EventGraph", includeAllEntryPoints:true}`
   → same error: `[NODE_NOT_FOUND] Could not find a starting node (event or function entry).`

3. Entry-points-only:
   `get_execution_flow {... graphName:"EventGraph", entryPointsOnly:true}`
   → same error: `[NODE_NOT_FOUND] Could not find a starting node (event or function entry).`

4. Same call WITH the Timeline node as the explicit start succeeds and even
   reports an empty entry-point set:
   `get_execution_flow {... graphName:"EventGraph", startNodeId:"AFB695704811711CF90345BB38C4C6BA"}`
   → `{"entryPoints":[], "executionChain":[{index:0, nodeType:"K2Node_Timeline", nodeTitle:"Bounce",
   execOutputs:[{pin:"Update", targetNodeTitle:"Set Relative Location"}, {pin:"Impact", targetNodeTitle:"Spawn Emitter at Location"}]}, ...], "nodeCount":4, ...}`

So the tool will happily root execution at the Timeline node — it just refuses
to discover it automatically, and the failure message gives no path forward.

**Workaround:** call `get_graph_details` / `find_nodes` to get the
`K2Node_Timeline` node's GUID, then re-call `get_execution_flow` with that GUID
as `startNodeId`.

**Fix (pick one or more):**
- In the entry-point collector (the shared `IsBlueprintEntryNode` / default-
  start logic used by `get_execution_flow`), treat latent event-producing
  nodes — `K2Node_Timeline` (auto-play or otherwise), and arguably
  `K2Node_InputAction`/timeline-like nodes — whose exec **output** pins root a
  chain as discoverable entry points when no `K2Node_Event`/`FunctionEntry`
  exists. A Timeline's `Update`/`Finished`/`Impact` outputs are entry roots.
- At minimum, make the failure actionable: when no event/function entry is
  found but the graph has exec-output-only nodes (timelines, etc.), return
  those candidate root node ids in the error payload (or a clean empty
  `entryPoints: []` result with a hint) instead of a bare `NODE_NOT_FOUND`, and
  document on the wiki page that timeline-rooted graphs need an explicit
  `startNodeId`.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed via mcp__editor-automation__call on `/Game/ExampleContent/Blueprints/Blueprints/BP_Timeline_Ball.BP_Timeline_Ball` (EventGraph, ubergraph, 10 nodes; only root is `K2Node_Timeline` "Bounce", auto-play, no event/function entry node). Default auto-start, `includeAllEntryPoints:true`, and `entryPointsOnly:true` all return `[NODE_NOT_FOUND] Could not find a starting node (event or function entry).`; the identical call with `startNodeId` = the Timeline node GUID succeeds, walks the full Update/Impact exec chain, and reports `entryPoints:[]`. The entry-point collector recognizes only event/function-entry nodes, never a latent event-producing Timeline whose exec-output pins root the graph; the error names no remedy. Distinct from B-bpir-entry-points-skip-composite-subgraphs (events nested in a K2Node_Composite — here there is no event node at all) and from E-find-nodes-eventgraph-default (find_nodes scoping). Seed method get_execution_flow.
- `#2-latent-exec-root-fallback` `IN-REVIEW` developer — Added a scoped latent-root fallback to `get_execution_flow` instead of broadening the shared entry collector (which the decompiler / orphan walker depend on). New shared helper `BlueprintHandlerUtils::CollectLatentExecRootNodes` (declared in `Private/Handlers/Blueprint/BlueprintHandlerUtils.h`, implemented in `BlueprintHandlerUtils.cpp`) collects exec-output-only roots: a node with a connected exec OUTPUT pin but no connected exec INPUT pin and that is not already an `IsBlueprintEntryNode` entry — class-agnostic, so an auto-play `K2Node_Timeline` (Update/Finished drive the chain, Play/Stop unwired) qualifies, while a timeline called from upstream (Play exec input wired) is deliberately excluded so it is not double-listed. `BlueprintGraphInspectionHandler.cpp::get_execution_flow` now calls this fallback only when `CollectEntryNodesRecursive` returns zero entries, feeding the latent roots into default-start selection, `entryPoints`, `entryPointsOnly`, and `includeAllEntryPoints` — so all three previously-failing modes resolve the Timeline. Result carries a new `entryPointsAreLatentRoots` bool; the residual `NODE_NOT_FOUND` message is now actionable (tells the caller to pass `startNodeId` found via find_nodes/get_graph_details). Ordinary event-rooted graphs are untouched (fallback never runs). Wiki: added a `### blueprint.graph.get_execution_flow` section to `docs/wiki-src/blueprint.graph.md` documenting timeline-rooted discovery and the latent-root semantics. Regression test: `Private/Tests/Blueprint/TestExecutionFlowTimelineRoot.cpp` builds an Actor BP whose EventGraph holds only a `create_node`-built Timeline wired Update→PrintString (zero event nodes), and asserts (a) the collector finds the Timeline and excludes the wired downstream node, (b) default auto-start succeeds (not NODE_NOT_FOUND) with the Timeline as start, `entryPointsAreLatentRoots:true`, and the chain reaching the downstream node, and (c) `entryPointsOnly` and `includeAllEntryPoints` both list the Timeline — every assertion flips to failure if the fallback is reverted.
</content>
</invoke>
