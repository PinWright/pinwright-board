---
id: B-bpir-entry-points-skip-composite-subgraphs
title: "Reachability collector skips events nested in K2Node_Composite collapsed graphs"
status: DONE
severity: Critical
category: bug
tags: [bpir, decompile, reachability, entry-points, k2node-composite, k2node-tunnel, collapsed-graph, orphan-detection, false-positive, blueprint-inspect, execution-flow]
---

# Reachability collector skips events nested in K2Node_Composite collapsed graphs

The shared entry-point collector used by `blueprint.decompile`,
`blueprint.inspect`, `blueprint.graph.get_execution_flow`, and
`blueprint.graph.find_orphaned_nodes` walks `K2Node_Event` only at the top
level of ubergraph and function graphs. It does **not** descend into
`K2Node_Composite::SubGraphs` to enumerate event nodes that tunnel exec out
via `K2Node_Tunnel "Outputs"` pins. This is an idiomatic UE pattern for
organising per-tick logic — `Event Tick`, `Event BeginPlay`, and
`Event Async Physics Tick` are placed inside a collapsed graph and the work
is exposed via tunnel outputs back to the parent graph.

When a Blueprint uses this pattern:

- The decompiler omits the events from "Reachable entry points".
- Every node downstream of the composite's tunnel outputs in the parent
  graph is reported as orphaned.
- Every internal body node of the composite is invisible to the orphan
  walker.

The four affected tools all return a confidently-wrong answer (no error,
no warning that the analysis may be incomplete). Downstream agents have
no signal to second-guess the result.

## Repro

Asset: `/App/HELIOS/Framework/HELIOS_BP`. The EventGraph contains a
`K2Node_Composite` named "Activation Node" at `(0,0)`. Inside it:
`K2Node_Event "Event Tick"`, `Event BeginPlay`, `Event Async Physics Tick`.
The body chain runs `Event Tick → Set DeltaSeconds →
UpdatePhysicsValues → Update Audio Activation →
Motors: Complete Audio & Visual System → Branch(IsLocallyControlled) →
Tunnel.OnTick`. A second composite "Camera HUB Node" holds
`Camera HUB: Complete System` similarly. All these systems run in-game
and produce visible/audible behaviour (motor SFX, propeller animation,
audio activation, boom-arm interp, FPVCamera HUD broadcasts).

Observed:

- `mcp__editor_automation__.call path="blueprint.graph.list_graphs" args={...}` → 23 graphs. **Inner composite subgraphs
  ("Activation Node", "Camera HUB Node", "MathExpression") are not in the
  list**, though they are addressable by name in `blueprint.graph.get_nodes`.
  Sibling discovery defect: agents have no path to learn these subgraphs
  exist.
- `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"Event Tick"}` (across all 23 listed
  graphs) → 0 hits.
- `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"ReceiveBeginPlay"}` → 0 hits.
- `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"K2Node_Event"}` → 6 events, none of
  Tick / BeginPlay / AsyncPhysicsTick.
- `mcp__editor_automation__.call path="blueprint.graph.get_execution_flow" args={"entryPointsOnly":true,"includeAllEntryPoints":true}` → 9 entries, exactly matching the
  decompiler's "Reachable entry points" list. Both tools agree, both
  wrong.
- `mcp__editor_automation__.call path="blueprint.graph.find_orphaned_nodes" args={...}` → 23 orphans, including the
  `K2Node_Composite "Activation Node"`, `K2Node_Composite
  "Camera HUB Node"`, `Sequence`, `Delay`, four reroute knots, the Lyra
  Verb Message broadcast cluster, `Set members in Camera Focus
  Settings`, MathExpression internals — all of which are reachable
  in-game.
- `mcp__editor_automation__.call path="blueprint.graph.get_node_details" args={...}` on the Activation Node composite →
  only Output exec pins (`On Begin Play`, `On Tick`, `On Async Tick`); no
  input exec. Confirms the reachability walker treats it as
  sink-only / unreachable.
- `mcp__editor_automation__.call path="blueprint.graph.get_nodes" args={"graphName":"Activation Node"}` → contains the
  three K2Node_Event entries plus the body chain. Once the inner graph
  name is known, individual node-level tools work correctly.

## Impact

A planning subagent in the session that surfaced this defect concluded
that motor SFX, propeller animation, audio activation, and boom-arm
interp on `HELIOS_BP` were all "dead code" because the decompiler showed
their callers as orphaned. Based on that conclusion, the agent
recommended deleting the BP graphs and replacing them with C++. The user
verified in-editor that all four subsystems work today, exposing the
false-positive.

Severity rationale (Critical):

- Wrong answer is silent (no warning, no error) — agents have no
  cross-check.
- Four inspection tools all share the underlying entry-point collector,
  so an agent's natural fallback ("ask a different MCP tool") returns the
  same wrong answer.
- The pattern (Event Tick inside a collapsed graph) is common in
  organised UE Blueprints — every BP that uses it will mis-report.
- Drove a planning agent to recommend deleting working subsystems.

The pattern shape is identical to
`B-integrity-gate-false-positive-asyncaction-proxies` (DONE) — whitelist
enumerated from the wrong abstraction layer — though that fix was scoped
to a different tool (`PluginIntegrityGate`).

**Workaround:** Once the inner subgraph name is guessed (e.g. "Activation
Node"), `mcp__editor_automation__.call path="blueprint.graph.get_nodes" args={"graphName":"<inner>"}` works. There is
no discovery path: `blueprint.graph.list_graphs` doesn't surface the
inner names. Agents would have to enumerate all `K2Node_Composite` nodes
in known graphs and walk their `BoundGraph` references manually.

**Fix:** In the entry-point collector (likely `BpirDecompiler.cpp` and
the shared reachability scanner under
`Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/`),
recurse into `UEdGraph::SubGraphs` whenever a `UK2Node_Composite` is
encountered. Treat any `K2Node_Event` found in a subgraph as an entry
point; follow exec chains across `K2Node_Tunnel` boundaries by mapping
the inner Tunnel "Outputs" pin name to the corresponding outer
`K2Node_Composite` output exec pin
(`UK2Node_Composite::OutputSourceNode` / `BoundGraph` provides the
link). Apply the same recursion to `find_orphaned_nodes` so internal
body nodes of composites are scanned alongside the parent graph.

Sibling discovery fix: `blueprint.graph.list_graphs` should also surface
inner composite subgraphs. Either flat with a `parentGraphName` field on
each entry, or as a nested `subGraphs` array per top-level graph. Same
treatment for `K2Node_MathExpression` subgraphs.

Regression test: minimal Blueprint with `Event Tick` inside a
`K2Node_Composite` tunnelling to `Print String`. Expectation: decompiler
"Reachable entry points" lists the tunnelled `Event Tick`;
`find_orphaned_nodes` returns zero orphans;
`blueprint.graph.get_execution_flow` with `entryPointsOnly:true` returns the
event as an entry; `blueprint.graph.list_graphs` includes the inner
composite subgraph by name.

## History
- `#1-initial-repro` `OPEN` reporter — Audit subagent on `/App/HELIOS/Framework/HELIOS_BP` reported `Motors: Complete Audio & Visual System`, `Camera HUB: Complete System`, `Camera HUB: Complete Setup`, and `Update Audio Activation` as dead code based on `blueprint.decompile` orphan warnings and `find_orphaned_nodes` results. User confirmed all four subsystems run in-game. Investigation traced the false-positive to `K2Node_Event` (`Event Tick`, `Event BeginPlay`, `Event Async Physics Tick`) being placed inside a `K2Node_Composite` "Activation Node" at top of EventGraph, with body tunnelling out via `K2Node_Tunnel "Outputs"` pins. Reachability collector does not descend into `K2Node_Composite::SubGraphs`. Affects `blueprint.decompile`, `blueprint.inspect`, `blueprint.graph.get_execution_flow`, `blueprint.graph.find_orphaned_nodes`. Sibling defect: `blueprint.graph.list_graphs` omits inner composite subgraph names so agents cannot discover them.
- `#2-fix-shipped` `IN-REVIEW` implementer — Promoted `IsBlueprintEntryNode` and `CollectAllBlueprintGraphsRecursive` to `BlueprintHandlerUtils` (Steps A). Deleted the duplicate `IsEntryNode` static functions in `GraphWalker.cpp` and `BlueprintGraphOrphanHandler.cpp` (and removed the "must stay in sync" comment). `GraphWalker::FindEntryPoints` now descends via `UEdGraph::GetAllChildrenGraphs` to collect entries from composite BoundGraphs (Step B). `BuildExecReachabilitySet` in the orphan handler seeds reachability from composite nodes whose BoundGraph contains an entry-point node, eliminating the false-positive chain (Step D). `get_execution_flow` inline lambda replaced by `IsBlueprintEntryNode` + child-graph descent (Step E). `list_graphs` appends subgraphs with `parentGraphName` field via `Blueprint->GetAllGraphs` (Step E). `BlueprintInspectHandler` includes nested subgraphs in the graphs array and tick scan (Step F). Four regression tests in `TestBlueprintGraphCompositeEntryPoints.cpp` verify orphan handler, decompiler, execution flow, and list_graphs through production RPC handlers.
- `#3-post-review-followups` `IN-REVIEW` implementer — Two follow-up rounds after spec/quality review: (a) `GraphWalker::FindEntryPoints` sort comparator now uses composite key `(GraphPathName, NodePosY)` for determinism across entries from different child graphs. (b) Added bidirectional WorkQueue bridge in `BlueprintGraphHandler::BuildExecutionChain` so BFS crosses `K2Node_Composite` boundaries — outer-into-composite via `InputSinkNode`'s output exec pins, inner-out-of-composite via the owning composite's matching output pins (pin-name equality). Fifth regression test `FCompositeEntryPointExecutionChainBridgeTest` guards the bridge by asserting an outer-graph `PrintString` reachable only via the bridge appears in the execution chain. (c) Wired `CollectAllBlueprintGraphsRecursive` into `BlueprintGraphOrphanHandler::CollectBlueprintGraphs`, `list_graphs` subgraph append, and `BlueprintInspectHandler` (single graph traversal reused for both JSON array and tick scan); helper kept inline in `GraphWalker::FindEntryPoints` and `get_execution_flow EntryNodes` since both operate on a single root `UEdGraph`. (d) Test fixture now creates `BoundGraph` with `CompositeNode` as outer (production-correct shape), allowing the speculative O(graphs × nodes) fallback scan inside `BuildExecutionChain` to be removed.
- `#4-verified-all-four-tools` `DONE` tester — Verified on the original repro asset `/App/HELIOS/Framework/HELIOS_BP`: (1) `blueprint.graph.list_graphs` returned 16 graphs including `Activation Node` and `Camera HUB Node` as `kind:"subgraph"` with `parentGraphName:"EventGraph"` (pre-fix these were absent — sibling discovery fix confirmed). (2) `blueprint.graph.find_orphaned_nodes` (all-graphs sweep, dryRun) returned `orphanedCount:0, totalNodes:222` across the same 16 graphs (pre-fix returned 23 orphans including Activation Node, Camera HUB Node, Sequence, Delay, etc.). (3) `blueprint.decompile_bpir graphName:"EventGraph"` emitted `entry event Tick`, `entry event BeginPlay`, `entry override ReceiveAsyncPhysicsTick` as proper top-level entries (the composite-tunnelled events) — pre-fix these were missing from the entry list. (4) The decompiler `warnings` array contained zero "Orphaned node" entries — sibling ticket `B-bpir-decompile-warnings-skip-composite-bridge` fix is also verified by this same run. All four affected tools agree on a clean reachability picture for composite-organised blueprints.
