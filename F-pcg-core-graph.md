---
id: F-pcg-core-graph
title: "PCG graph create + node/edge authoring (new namespace)"
status: DONE
severity: High
category: feature
tags: [pcg, graph, rpc, authoring, namespace]
---

# PCG Graph Create + Node/Edge Authoring

The gateway exposes no `pcg.*` namespace. Procedural Content Generation
(`UPCGGraph` / `UPCGNode`, modules `PCG` + `PCGEditor`) is a first-class UE5
authoring surface and follows the same `UEdGraph`-style ownership as Niagara
and Blueprint, so the existing handler pattern (auto-registration,
`FHandlerContext`, `FPluginState` tracking, deferred-during-GC dispatch)
transfers cleanly.

Add an imperative core for the namespace:

- `pcg.create_graph` — create a new `UPCGGraph` asset at a content path.
- `pcg.add_node` — append a node by class path (e.g.
  `/Script/PCG.PCGMeshSpawnerSettings`), with required `x`/`y` coordinates
  per the project's no-auto-layout rule for imperative per-node RPCs.
- `pcg.connect_pins` — wire a source output pin to a target input pin by
  node id + pin name.
- `pcg.remove_node` — delete a node and its incident edges.
- `pcg.inspect` — read-first dump: graph metadata, node list with class
  paths and settings, edges with pin names. Mirrors `niagara.inspect` and
  `blueprint.inspect` in shape.

Scope deliberately excludes typed convenience wrappers (mesh spawner,
surface sampler, etc.) — the generic `add_node` is the foundation, and
the Niagara / MGIR precedent shows typed shims should wait for concrete
demand rather than being pre-allocated.

**Open question (deferred):** Text IR (PCGIR) is out of scope for this
ticket. The MGIR-after-Niagara precedent shows IR is justified only after
imperative RPCs ship and users hit per-node-call friction at scale. File
a separate `F-pcgir-*` ticket if that friction materializes.

**Fix:** New `Source/EditorAutomationRpcGateway/Private/Handlers/PCG/`
domain folder. Add `PCG` and `PCGEditor` to `Build.cs` via
`TryAddConditionalModule()` (the modules are not present in every UE
install / project). Mirror Niagara handler organization
(`PCGGraphCreate.cpp`, `PCGGraphAuthoring.cpp`, `PCGGraphInspect.cpp`).
Track open PCG graph editors in `FPluginState` if mutation requires it.
Add focused tests under `Source/EditorAutomationRpcGatewayTests/` covering
create → add_node → connect_pins → inspect → remove_node round-trip.
Add `docs/wiki/pcg.md` overlay describing the namespace and class-path
resolution conventions.

## History
- `#1-pcg-namespace-request` `OPEN` reporter — Filed: no `pcg.*` namespace exists in the gateway. Verified absence by grepping `rpc-method-reference.generated.md`, `Source/EditorAutomationRpcGateway/Private/Handlers/`, and `docs/board/`. Scoped to imperative core RPCs only (`create_graph`, `add_node`, `connect_pins`, `remove_node`, `inspect`); split typed-helper variants off the original proposal as YAGNI per Niagara/MGIR precedent; deferred PCGIR text-IR pending real demand. Broad UE5 dev audience — PCG is engine-standard, not PDS-specific.
- `#2-implemented-pcg-namespace` `IN-REVIEW` developer — Added `pcg.create_graph`, `pcg.add_node`, `pcg.connect_pins`, `pcg.remove_node`, `pcg.inspect` in `Handlers/PCG/` (PCGGraphCreate.cpp, PCGGraphAuthoring.cpp, PCGGraphInspect.cpp). All files guarded by `#if __has_include("PCGGraph.h")` so the plugin still builds on UE installs without PCG enabled. `pcg.add_node` requires `x` and `y` per the project's no-auto-layout rule. `pcg.remove_node` refuses to delete `GetInputNode()`/`GetOutputNode()` with `IMMUTABLE_NODE`. Build.cs adds `PCGEditor` (PCG runtime was already wired at line 91). Wiki overlay added at `docs/wiki/pcg.md`. Tests in `Tests/PCG/TestPCGGraphHandlers.cpp` cover asset creation, add-node increment, and immutable-node refusal; counterfactual: reverting AddNodeOfType makes GetNodes().Num() stay at 0.
- `#3-connect-pins-cannot-resolve-nodes` `OPEN` tester — Returned: `pcg.connect_pins` fails with `NODE_NOT_FOUND` ("Could not find node(s): from='CreatePoints_0' to='DefaultOutputNode'") even though `pcg.inspect` called against the same graph in the same batch lists both ids verbatim. The other 4 RPCs work: `create_graph` produced `/Game/App/UI/Test/PCG_McpVerifyTemp_FpcgCoreGraph2` with `assetClass: PCGGraph`; `add_node` returned `nodeId: CreatePoints_0`; `inspect` returned that node plus DefaultInput/DefaultOutput with pins; `remove_node` deleted `CreatePoints_0` cleanly in an earlier batch and refused `DefaultInputNode` with `IMMUTABLE_NODE` as designed. Test: batch `[create_graph → add_node → connect_pins(fromNode:"CreatePoints_0", fromPin:"Out", toNode:"DefaultOutputNode", toPin:"Out") → inspect → asset.delete]` against `http://127.0.0.1:19880/rpc`. Inspect immediately after the failed connect shows `edges: []` and both nodes present, so the resolver in PCGGraphAuthoring.cpp is the suspect — likely mismatched lookup (GetFName vs GetName, or it's not consulting the implicit input/output nodes the way inspect does). Temp assets cleaned up.
- `#4-resolve-implicit-connect-nodes` `IN-REVIEW` developer — Changed PCG node resolution for pcg.connect_pins to include UPCGGraph::GetInputNode() / GetOutputNode() when resolving node ids returned by pcg.inspect, while keeping user-node-only resolution for typed node mutations. Added FPCGConnectPinsResolvesImplicitOutputNodeTest covering implicit output endpoint edge creation; counterfactual: reverting the implicit-node resolver makes pcg.connect_pins return NODE_NOT_FOUND and leaves the graph with no edge.
- `#5-verify-fix` `DONE` tester — Verified: reran the exact #3 repro batch against http://127.0.0.1:19880/rpc — `create_graph(name:"PCG_McpVerifyTemp_FpcgCoreGraph", savePath:"/Game/App/UI/Test")` → `add_node` returned `nodeId:"CreatePoints_0"` → `connect_pins(fromNode:"CreatePoints_0", fromPin:"Out", toNode:"DefaultOutputNode", toPin:"Out")` now returns `{connected:true}` (was `NODE_NOT_FOUND`). Subsequent `inspect` shows `edges:[{from:"DefaultOutputNode",fromPin:"Out",to:"CreatePoints_0",toPin:"Out"}]` confirming the implicit output endpoint resolved. Temp asset deleted via `asset.delete{path:...}`.
