---
id: B-pcg-inspect-edge-direction-reversed
title: "pcg.inspect reports every edge's from/to (and fromPin/toPin) reversed"
status: IN-REVIEW
severity: High
category: bug
tags: [pcg, inspect, edges, topology, output-schema]
---

# `pcg.inspect` reports edge direction backwards

`pcg.inspect` returns an `edges[]` array where each edge's `from`/`fromPin`
and `to`/`toPin` are **swapped** relative to the actual data-flow direction.
Every reported edge points upstream when it should point downstream: the
`from`/`fromPin` fields name the **destination** node + its *input* pin, and
the `to`/`toPin` fields name the **source** node + its *output* pin. This
makes the topology dump unusable for confirming wiring — the canonical reason
to call `pcg.inspect` after authoring a graph.

## Root cause

`Private/Handlers/PCG/PCGGraphInspect.cpp` (the `AppendNodeEdges` lambda)
reads the two `UPCGEdge` pins with the roles inverted:

```cpp
const UPCGNode* SrcNode = Edge->OutputPin->Node;   // wrong: OutputPin is the DOWNSTREAM end
const UPCGNode* DstNode = Edge->InputPin->Node;    // wrong: InputPin is the UPSTREAM end
EdgeObj->SetStringField(TEXT("from"), SrcNode->GetName());
EdgeObj->SetStringField(TEXT("fromPin"), Edge->OutputPin->Properties.Label.ToString());
EdgeObj->SetStringField(TEXT("to"), DstNode->GetName());
EdgeObj->SetStringField(TEXT("toPin"), Edge->InputPin->Properties.Label.ToString());
```

The accompanying code comment ("Edge::OutputPin is the source-side pin (this
node's output); Edge::InputPin is the destination-side pin") is the inverted
mental model that produced the bug, and it directly contradicts the engine
header. `Engine/Plugins/PCG/Source/PCG/Public/PCGEdge.h` documents:

```cpp
/** Pin at upstream end of edge. */
TObjectPtr<UPCGPin> InputPin;
/** Pin at downstream end of edge. */
TObjectPtr<UPCGPin> OutputPin;
```

i.e. `Edge->InputPin` is the **upstream/source** pin and `Edge->OutputPin` is
the **downstream/destination** pin — the opposite of what the handler assumes.

The plugin's own PCGIR decompiler gets this right and can serve as the
reference (`Private/PCGIR/PCGIRDecompiler.cpp`):

```cpp
// UPCGEdge::InputPin is the upstream (source/output) pin;
// UPCGEdge::OutputPin is the downstream (dest/input) pin.
UPCGPin* SourcePin = Edge->InputPin;
UPCGPin* DestPin   = Edge->OutputPin;
```

So `pcg.inspect` and `pcg.pcgir` disagree on edge direction within the same
codebase; `pcg.inspect` is the wrong one.

## Verbatim repro

Graph `/Game/PCG/RockScatter` was authored with the chain
`DefaultInputNode.In(out) -> SurfaceSampler_0.Surface(in)`,
`SurfaceSampler_0.Out -> TransformPoints_0.In`,
`TransformPoints_0.Out -> Spatial Noise_0.In`,
`Spatial Noise_0.Out -> SelfPruning_0.In`,
`SelfPruning_0.Out -> DefaultOutputNode.Out(in)` — all five via
`pcg.connect_pins` with `{connected:true}`.

`call("pcg.inspect", {graphPath:"/Game/PCG/RockScatter"})` returns (edges only,
verbatim):

```json
"edges":[
 {"from":"SurfaceSampler_0","fromPin":"Surface","to":"DefaultInputNode","toPin":"In"},
 {"from":"TransformPoints_0","fromPin":"In","to":"SurfaceSampler_0","toPin":"Out"},
 {"from":"Spatial Noise_0","fromPin":"In","to":"TransformPoints_0","toPin":"Out"},
 {"from":"SelfPruning_0","fromPin":"In","to":"Spatial Noise_0","toPin":"Out"},
 {"from":"DefaultOutputNode","fromPin":"Out","to":"SelfPruning_0","toPin":"Out"}
]
```

Every row is reversed. Self-evident from the pin labels in the same dump:
`fromPin:"Surface"` is listed under `SurfaceSampler_0`'s `inputPins`, and
`toPin:"In"` (on `DefaultInputNode`) is that node's `outputPins` entry — so the
edge claims data flows from an input pin into an output pin, which is backwards.
The correct first row is
`{"from":"DefaultInputNode","fromPin":"In","to":"SurfaceSampler_0","toPin":"Surface"}`.

Deterministic: replayed `pcg.inspect` twice, byte-identical reversed output
both times.

This was latently visible (but unflagged) in `F-pcg-core-graph` history
`#5-verify-fix`: that tester wired `CreatePoints_0.Out -> DefaultOutputNode.Out`
and inspect reported `from:"DefaultOutputNode" ... to:"CreatePoints_0"` —
already reversed — but it was read as "confirming the edge resolved" without
noticing the direction was backwards.

## Fix

In `PCGGraphInspect.cpp::AppendNodeEdges`, swap the pin roles to match the
engine semantics and the PCGIR decompiler: `from`/`fromPin` come from
`Edge->InputPin` (upstream), `to`/`toPin` come from `Edge->OutputPin`
(downstream). Fix the misleading code comment in the same edit. Add a test
that asserts edge **direction** (not just edge existence) on a known
input->node->output chain.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed against `/Game/PCG/RockScatter` via `mcp__editor-automation__call`: `pcg.inspect` returns all five edges with `from`/`to` and `fromPin`/`toPin` swapped vs. the wiring issued by `pcg.connect_pins` (self-evident — `fromPin` names input pins, `toPin` names output pins). Root cause in `PCGGraphInspect.cpp`: it reads `Edge->OutputPin` as source and `Edge->InputPin` as dest, but `PCGEdge.h` documents `InputPin`=upstream / `OutputPin`=downstream; the sibling `PCGIRDecompiler.cpp` has it correct. Deterministic across two replays. Deduped: no existing `B-pcg*` ticket; the DONE PCG feature tickets (`F-pcg-core-graph`, `F-pcg-filters-and-subgraphs`) never asserted edge direction.
- `#2-fix` `IN-REVIEW` developer — Fixed in `Private/Handlers/PCG/PCGGraphInspect.cpp` (`AppendNodeEdges` lambda): swapped the pin roles so `from`/`fromPin` now come from `Edge->InputPin` (upstream/source) and `to`/`toPin` from `Edge->OutputPin` (downstream/dest), matching `PCGEdge.h` and the sibling `PCGIRDecompiler.cpp`; rewrote the misleading code comment to state the correct InputPin=upstream / OutputPin=downstream semantics. Added regression test `FPCGInspectReportsEdgeDirectionTest` (`EditorAutomationRpcGateway.pcg.inspect.EdgeDirection`) in `Private/Tests/PCG/TestPCGGraphHandlers.cpp`: builds a real source-output -> output-node-input edge via `UPCGPin::AddEdgeTo`, invokes the production `pcg.inspect` handler, and asserts the emitted edge JSON reports `from`=source / `to`=output-node / `fromPin`=`Out` / `toPin`=`In` — it fails (no matching edge found) if the pin-role swap is reverted. Existing coverage only asserted edge existence (`HasEdgeBetweenPins`), never direction, so this fills the gap. Not compiled/tested here (later phase).
