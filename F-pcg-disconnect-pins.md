---
id: F-pcg-disconnect-pins
title: "The pcg namespace has add_node/remove_node but connect_pins with no inverse — the only ways to break an edge are destroying the node that owns it or silently overwriting a single-connection pin, while the engine's precise single-edge disconnect is Blueprint-callable and one wrapper away"
status: OPEN
severity: Medium
category: feature
tags: [pcg, connect-pins, disconnect, remove-edge, missing-verb, authoring, graph-editing, symmetry, procedural-vegetation]
encounters: 1
lastSeen: 2026-08-29T17:20:00+05:00
---

# One authoring pair in this namespace is missing its other half

`pcg` ships 14 verbs. `add_node` has `remove_node`. `add_graph_parameter` has
`remove_graph_parameter`. `connect_pins` has nothing.

## The gap, verified

`grep -rni "disconnect|remove_edge|RemoveEdge|BreakEdge|BreakAllEdges|BreakAllIncompatibleEdges"`
over the entire `Source/PinWrightPCG/` tree, **tests included**: zero hits. The complete registered
set is `add_noise_filter`, `add_slope_filter`, `add_subgraph`, `decompile`, `generate`, `add_node`,
`connect_pins`, `remove_node`, `create_graph`, `inspect`, `add_graph_parameter`,
`list_graph_parameters`, `remove_graph_parameter`, `set_self_pruning_settings`.

The only edge removal available is **indirect and destructive**: `pcg.remove_node` calls
`Graph->RemoveNode(Node)` (`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphAuthoring.cpp:198`),
and its own registered summary at `:165` says what that costs — *"Delete a node from a UPCGGraph
(and its incident edges)"*. There is no verb that removes an edge while leaving both nodes standing.

## Correction to the premise this was filed from

The lead read *"an edge into a multi-connection pin is unremovable through PinWright"*. That is half
right, and the accurate half matters for both the severity and the fix.

Unreachable through any **verb**: yes. Unreachable outright: **no.** The engine's primitive exists,
is precise, and is Blueprint-exposed — so `python.execute` (or `object.call_function`) reaches it
today.

`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Public/PCGGraph.h:479-481`:

```cpp
/** Removes an edge in the graph. Returns true if an edge was removed. */
UFUNCTION(BlueprintCallable, Category = Graph)
UE_API bool RemoveEdge(UPCGNode* From, const FName& FromLabel, UPCGNode* To, const FName& ToLabel);
```

Defined at `PCGGraph.cpp:1657-1691`: it resolves both pins (`:1669-1670`), calls
`OutPin->BreakEdgeTo(InPin, &TouchedNodes)` (`:1675`), and returns `TouchedNodes.Num() > 0`
(`:1690`) — a single-edge disconnect with a truthful return value, already written.

Two coarser siblings exist and are **C++-only** (no `UFUNCTION`): `RemoveInboundEdges` and
`RemoveOutboundEdges` (`PCGGraph.h:531-532`, defined `PCGGraph.cpp:1747-1774` and `:1776-1803`),
which call `BreakAllEdges` on the named pin (`:1758`, `:1788`) — all-or-nothing per pin. There is no
`BreakEdge` member on `UPCGGraph`. Pin-level primitives also exist: `UPCGPin::BreakEdgeTo`
(`PCGPin.h:301`, `PCGPin.cpp:206`) and `UPCGPin::BreakAllEdges` (`PCGPin.h:304`, `PCGPin.cpp:240`).

So this is not "the capability does not exist". It is **"the capability exists, is one line, and is
only reachable by leaving the plugin"** — which is the shape the board treats as a gap to close
rather than a blocker, and it is why this is filed Medium rather than High.

## Why the absence still costs real work

The two routes available through typed verbs are both bad, and for one pin shape neither applies.

**Destroying the node is not a substitute.** `pcg.remove_node` takes every property on that node with
it. A Procedural Vegetation node carries deeply nested settings structs, and `F-property-set-batch`
records the going rate: one PCG graph cost **~110 individual `property.set` calls**. Re-adding a node
to change one wire means paying that again, and there is no batch write to pay it with.

**Overwriting is not a substitute; it is a defect.** On a single-connection pin you can displace the
occupant by connecting something else — which is exactly the silent destruction filed as
`B-pcg-connect-pins-silently-replaces-edge`. Using it deliberately as a disconnect means relying on
behaviour that reports nothing, and it can only ever *replace* an edge, never leave the pin empty.

**On a multi-connection pin neither applies.** Nothing displaces an edge there, so the typed-verb
options collapse to `remove_node` alone.

## Proposed shape

Mirror `connect_pins` exactly, so the pair reads as a pair:

    pcg.disconnect_pins(graphPath, fromNode, fromPin, toNode, toPin)

- Resolve nodes and pins with the handler's existing helpers — `FindNodeByNameIncludingImplicit` and
  `FindPinByLabel`, already used at `PCGGraphAuthoring.cpp:130-149` — so endpoint nodes
  (`GetInputNode()` / `GetOutputNode()`) work the same way they do on `connect_pins`, and the same
  `NODE_NOT_FOUND` / `PIN_NOT_FOUND` errors apply.
- Call `UPCGGraph::RemoveEdge` and **publish its return value**, never a literal. `EDGE_NOT_FOUND`
  when both pins resolved and the bool is false. This is deliberately the opposite of what
  `connect_pins` does today (`PCGGraphAuthoring.cpp:155` writes a constant `true`), and a fixer
  landing both should make the pair symmetric in honesty as well as in shape — see
  `B-pcg-connect-pins-silently-replaces-edge`, whose fix is the same one-line discipline on the
  other side.
- `MarkPackageDirty` as the siblings do (`:152`), and wrap the mutation in an `FScopedTransaction` —
  which none of this file's existing mutating verbs do, a namespace-wide gap recorded on
  `B-pcg-connect-pins-silently-replaces-edge` and not owned here.

A pin-scoped variant (`disconnect all edges on this pin`) maps to `RemoveInboundEdges` /
`RemoveOutboundEdges` and is worth adding only if a caller asks; the single-edge form is the one the
absence actually bites, and shipping both at once would be scope this ticket cannot justify.

## Related

- **`B-pcg-connect-pins-silently-replaces-edge`** (OPEN, High, filed this session) — the reason this
  ticket is load-bearing rather than a symmetry complaint. That defect destroys an edge without
  saying so; this gap is why the destruction is expensive to undo. Read together: they are the two
  halves of one authoring story, but they are independently valuable and independently testable —
  the truthful `replacedExistingEdge` bit is worth having with no disconnect verb, and a disconnect
  verb is worth having even if `connect_pins` starts telling the truth — so they stay separate per
  the no-umbrella policy.
- `F-pcg-authoring-parity` (IN-REVIEW, Medium) — the ticket that established this namespace's
  discrete-per-capability cadence and split three capability asks out of an audit umbrella. This is
  the same cadence: one verb, one acceptance test.
- `F-pcg-core-graph` (DONE) — shipped `connect_pins` and `remove_node`. Its scope line at `:27`
  reads *"`pcg.remove_node` — delete a node and its incident edges"*, and its round-trip acceptance
  at `:49` is `create → add_node → connect_pins → inspect → remove_node`. So the missing inverse was
  not overlooked in review; the round trip was closed through node deletion, which is precisely the
  substitution this ticket argues is too expensive now that graphs are being authored incrementally.
- `F-property-set-batch` (OPEN, Medium) — the reason `remove_node`-and-rebuild is not a cheap
  workaround. Referenced, not restated.

## Dedup

Board-wide search for `disconnect`, `remove_edge`, `RemoveEdge`, `break edge` and `connect_pins`
tickets across all namespaces. Nothing asks for edge removal in `pcg`. The `niagara.*` namespace has
its own `connect_pins` tickets (`B-niagara-connect-pins-sends-no-graph-notification`,
`E-niagara-connect-pins-add-pin-opaque`) — a different handler over a different engine API, no
shared code and no shared fix. `B-pcg-inspect-edge-direction-reversed` is the edge *read* path.
`F-bp-graph-replace-node-rpc` is the Blueprint namespace. Nothing overlaps.

## Severity

**Medium, argued.** Impact class is the rubric's *"High or Medium: hard blocker with no workaround
(a stub, a **missing verb**, or rejecting valid input)"* — a missing verb, and the typed-verb
alternatives are a destructive node deletion or a defect. It lands on **Medium** rather than High
for one reason, stated so a reviewer can disagree with it directly: `UPCGGraph::RemoveEdge` is
`UFUNCTION(BlueprintCallable)` (`PCGGraph.h:480-481`), so `python.execute` reaches the exact
primitive with the exact semantics, which makes this the rubric's Medium — *"doable, but only via a
documented workaround, a source dive, or many extra calls"* — rather than an absolute block. The
source dive is real (nothing in the wiki tells a caller that `RemoveEdge` exists or is reachable),
but it terminates in a working one-liner.

**The High reading, and why it is refused.** Combined with `B-pcg-connect-pins-silently-replaces-edge`
a caller can lose a wire and have no typed way to restore it, which reads as a hard block on
incremental graph authoring. That severity belongs to **that** ticket, which is filed High and owns
the silent loss; charging it again here would double-count one problem across two tickets on a board
whose picker ranks by severity — the same double-counting `F-generic-volume-creator-with-brush-geometry`
declined against `B-spawned-volumes-have-no-brush-geometry`. This ticket asks for a *capability*, not
for a lie to stop.

**Reach modifier declined in both directions, and named.** No bump up: graph authoring is common
enough that the namespace carries 14 verbs, but no one of them — and certainly not edge deletion,
which only arises when you are editing a graph rather than building one — runs in almost every
session. No bump down: this is not a rare edge path either, since every non-trivial PV graph is
built incrementally and every incremental build eventually rewires something. Medium stands
unmodified.

## History
- `#1-connect-pins-has-no-inverse` `OPEN` reporter — Filed from this session's Procedural Vegetation pass. Gap verified by grep over the whole `Source/PinWrightPCG/` tree including tests for `disconnect|remove_edge|RemoveEdge|BreakEdge|BreakAllEdges|BreakAllIncompatibleEdges` — **zero hits** — against the full registered set of 14 `pcg.*` verbs; the only edge removal is `pcg.remove_node`, which destroys the node (`PCGGraphAuthoring.cpp:198`, summary at `:165`: *"Delete a node from a UPCGGraph (and its incident edges)"*). **Corrects the premise it was filed from.** The lead said an edge into a multi-connection pin is *unremovable* through PinWright; it is unreachable through any verb, but `UPCGGraph::RemoveEdge` is `UFUNCTION(BlueprintCallable)` (`C:/UE_5.8/.../PCG/Public/PCGGraph.h:479-481`, defined `PCGGraph.cpp:1657-1691`, resolving pins at `:1669-1670`, `BreakEdgeTo` at `:1675`, returning `TouchedNodes.Num() > 0` at `:1690`), so `python.execute` reaches the exact primitive with the exact semantics. That correction is what holds this at Medium instead of High, and it also improves the ask: the verb is a one-to-one wrapper over an engine function that already returns the right bool. Recorded that the coarser `RemoveInboundEdges` / `RemoveOutboundEdges` (`PCGGraph.h:531-532`, `.cpp:1747-1774`/`:1776-1803`, `BreakAllEdges` at `:1758`/`:1788`) carry no `UFUNCTION` and are C++-only, and that no `BreakEdge` member exists on `UPCGGraph`. Why the gap still costs: `remove_node`-and-rebuild forfeits every property on the node, at the rate `F-property-set-batch` measured (~110 `property.set` calls for one PCG graph) with no batch write to pay it with; deliberate overwriting works only on a single-connection pin and is the silent destruction filed as `B-pcg-connect-pins-silently-replaces-edge`; and on a multi-connection pin neither applies. Proposed `pcg.disconnect_pins(graphPath, fromNode, fromPin, toNode, toPin)` mirroring `connect_pins`' five parameters and reusing `FindNodeByNameIncludingImplicit` / `FindPinByLabel` (`PCGGraphAuthoring.cpp:130-149`) so endpoint nodes and the `NODE_NOT_FOUND` / `PIN_NOT_FOUND` contract carry over, publishing `RemoveEdge`'s return value rather than a literal — deliberately the opposite of `connect_pins`' constant `true` at `:155`. Dedup: nothing on the board asks for edge removal in `pcg`; the `niagara.*` `connect_pins` tickets share no code; `B-pcg-inspect-edge-direction-reversed` is the read path. Noted that `F-pcg-core-graph`'s round-trip acceptance (`:49`) closed the loop through `remove_node`, so the missing inverse was a scope decision rather than an oversight — one that has become expensive now that graphs are authored incrementally. Severity Medium with the High reading named and refused, because the silent-loss severity belongs to `B-pcg-connect-pins-silently-replaces-edge` and double-counting it would mis-order a severity-ranked picker; reach declined in both directions.
