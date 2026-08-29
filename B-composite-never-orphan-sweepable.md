---
id: B-composite-never-orphan-sweepable
title: "A dead K2Node_Composite is never reported as an orphan: the bound-graph entry seed is a tautology for every well-formed composite"
status: OPEN
severity: Medium
category: bug
tags: [orphan-detection, find-orphaned-nodes, delete-orphaned-nodes, reachability, k2node-composite, collapsed-graph, math-expression, k2node-tunnel, false-negative, decompiler]
blockedBy: [B-orphan-sweep-mathexpression-fatal-load]
encounters: 1
lastSeen: 2026-08-29T00:00:00Z
---

# A dead collapsed graph is permanently unsweepable

Split out of `B-orphan-sweep-mathexpression-fatal-load` (#4). That ticket fixed the
half of the composite blind spot that *destroyed* data; this is the half that
*preserves* it forever. Filed separately because the fix touches a different
function and carries a materially different blast radius.

## Verified mechanism (current HEAD, plugin `b4413416`)

`BuildExecReachabilitySet` (`BlueprintHandlerUtils.cpp:2413`) seeds its BFS from two
kinds of node. The second arm, at `:2453`:

```cpp
else if (UK2Node_Composite* Composite = Cast<UK2Node_Composite>(Node))
{
    TArray<UEdGraphNode*> BoundEntries;
    CollectEntryNodesRecursive(Composite->BoundGraph, BoundEntries);
    if (BoundEntries.Num() > 0)
    {
        MarkNode(Composite, Queue);
    }
}
```

`CollectEntryNodesRecursive` (`:2345`) counts anything `IsBlueprintEntryNode` (`:2201`)
accepts, and that predicate's last arm is `IsMacroEntryTunnel` (`:2246`) — a bare
`UK2Node_Tunnel` with `bCanHaveOutputs && !bCanHaveInputs`. A composite's bound graph
always contains exactly that node: `UK2Node_Composite::PostPlacedNewNode`
(`C:\UE_5.8\Engine\Source\Editor\BlueprintGraph\Private\K2Node_Composite.cpp:310-338`)
creates it unconditionally with `bCanHaveOutputs = true; bCanHaveInputs = false`, and the
collapse/expand re-pairing paths (`K2Node_Composite.cpp:138-146`, `:191-199`) restore the
same shape. So for every **well-formed** composite `BoundEntries.Num() > 0` is a
tautology, and the branch does not mean what it reads as. It is not "seed composites whose
bound graph has an entry" — it is **"seed every composite"**, decided by a test that can
only return false when the composite is already *corrupt* (null `BoundGraph`, or an entry
tunnel that was destroyed). The predicate therefore selects precisely the wrong population:
healthy dead composites never, broken composites always.

Consequences that follow directly:

- `ScanGraphForBlueprintOrphans` (`:2580`) computes `bIsOrphan = !ReachableNodes.Contains(...)`,
  so a `UK2Node_Composite` / `UK2Node_MathExpression` is **never** emitted as an orphan
  descriptor no matter how dead it is — no exec input wired, no output consumed, nothing.
- After the Critical's fix the whole composite family is now closed to the sweep: the outer
  node is unconditionally reachable (`:2453`), the entry tunnel is skipped by
  `IsBlueprintEntryNode` (`:2601`), the exit tunnel is skipped by `IsMacroExitTunnel`
  (`:2611`), and the bound graph's inner nodes are reachable because they hang off the entry
  tunnel's exec seed. Correct as damage control, wrong as a sweep.
- The blind spot is **transitive upstream**. `BuildFullReachabilitySet` (`:2522`) pushes
  every already-reachable node into the backward data queue, so the dead composite anchors
  its entire input data chain as reachable. An arbitrarily large tree of pure feeder nodes
  whose only consumer is the dead composite is kept alive with it.

Not stale: this describes source at HEAD, after `#4-boundary-tunnel-never-swept` landed.
The Critical's own fix commentary states the same finding as an unfixed follow-up.

## Consequence: ergonomic, not correctness

A dead collapsed graph that is never removed is **dead weight, not damage**. Priced at each
stage:

- **At load** — real but small. The bound graph's exports load and `ReconstructAllNodes` runs
  over them on regenerate-on-load; for a `UK2Node_MathExpression` that is a full expression
  re-parse and node regeneration per cold load. Nothing fails.
- **At compile** — effectively free. `FKismetCompilerContext::PruneIsolatedNodes` runs at
  `KismetCompiler.cpp:3836`, *before* `ExpandTunnelsAndMacros` at `:3846`, so an isolated
  composite is pruned out of the merged ubergraph before it is ever expanded. No bytecode,
  no validation pass, no error, no warning.
- **In a dump / decompile** — where it actually costs. The BPIR orphan-warning pass
  (`BpirDecompiler.cpp:910`) builds its reachable set from the same
  `BuildExecReachabilitySet`, so a dead composite never produces an
  `Orphaned node not reachable from any entry point` warning either. A dead collapsed
  subgraph reads as live logic in every `blueprint.decompile` and every `asset.dump` bpir
  sidecar, and an agent reading the dump gets no signal that it is dead.
- **In the sweep's contract** — `find_orphaned_nodes` returning `orphanedCount: 0` on a graph
  that visibly contains a dead collapsed node is a silent false-negative. It is an omission,
  not a lie: nothing downstream is corrupted by trusting it, and no data is lost.

Verdict: **ergonomic**, at the "silently incomplete readback" end of it. The workaround is
exact and cheap — `blueprint.graph.delete_node` on the composite's guid, whose
`UK2Node_Composite::DestroyNode` already tears the bound graph down as a unit via
`FBlueprintEditorUtils::RemoveGraph`. Medium-band soft blocker, not a High-band
false-success.

**Workaround:** identify the dead composite by eye (or from `get_graph_details`) and call
`blueprint.graph.delete_node` on it directly. Do **not** delete its tunnels — see
`B-orphan-sweep-mathexpression-fatal-load`.

## Blast radius of a fix

The tempting one-line fix is to drop the `IsMacroEntryTunnel` arm from
`IsBlueprintEntryNode`. Below is every consumer that would change if the *shared*
entry-node definition moved. This is why that fix is not on the table.

### Direct `IsBlueprintEntryNode` call sites (7)

| # | Site | Effect of dropping the tunnel arm |
|---|------|-----------------------------------|
| 1 | `BlueprintHandlerUtils.cpp:2367` — inside `CollectEntryNodesRecursive` | Macro graphs and composite bound graphs lose their only entry, poisoning every consumer of `CollectEntryNodesRecursive` below. |
| 2 | `BlueprintHandlerUtils.cpp:2394` — `CollectLatentExecRootNodes` skip-list | An entry tunnel is *exactly* the latent-root shape (exec output connected, exec input unconnected). It would newly qualify, so `get_execution_flow`'s fallback would start reporting tunnels as latent exec roots. |
| 3 | `BlueprintHandlerUtils.cpp:2449` — seed loop in `BuildExecReachabilitySet` | Macro graph bodies lose their exec seed entirely. **Every node in every macro graph becomes an orphan** — a regression of the exact class the Critical just fixed, on a far wider population. |
| 4 | `BlueprintHandlerUtils.cpp:2456` — composite seed, via `CollectEntryNodesRecursive` | The defect site. The only place the change would be *wanted*. |
| 5 | `BlueprintHandlerUtils.cpp:2601` — `ScanGraphForBlueprintOrphans` skip | Entry tunnels become sweepable. `UK2Node_Tunnel::DestroyNode` (`K2Node_Tunnel.cpp:51-58`) nulls the twin's `InputSinkNode`, and `UK2Node_Composite::GetEntryNode()`'s `check(InputSinkNode)` (`K2Node_Composite.cpp:344`) is the exact mirror of the fatal assert in `B-orphan-sweep-mathexpression-fatal-load`. **Reintroduces a Critical.** |
| 6 | `Decompiler/BpirDecompiler.cpp:855` — unvisited-node statement pass | Entry tunnels start emitting as standalone BPIR statements inside macro and composite bodies. |
| 7 | `Decompiler/BpirDecompiler.cpp:933` — orphan-warning pass | Every macro-graph dump gains a spurious `Orphaned node not reachable from any entry point` warning for its entry tunnel. |

### Direct `CollectEntryNodesRecursive` call sites (2)

| # | Site | Effect |
|---|------|--------|
| 8 | `Decompiler/GraphWalker.cpp:127` — `FGraphWalker::FindEntryPoints`, consumed by `BpirDecompiler.cpp:622` | Macro graphs decompile to nothing: zero entries routes to the empty-graph marker path, so `blueprint.decompile` and every `asset.dump` bpir sidecar for a macro-bearing Blueprint silently loses content. |
| 9 | `Handlers/Blueprint/BlueprintGraphInspectionHandler.cpp:1373` — `blueprint.graph.get_execution_flow` | With no explicit `startNodeId`, a macro graph collects zero entries, falls through to `CollectLatentExecRootNodes` (which per #2 would then hand back the tunnel), and otherwise returns bare `NODE_NOT_FOUND`. |

`BlueprintHandlerUtils.cpp:2456` is the self-recursive descent inside
`BuildExecReachabilitySet` and is counted as #4.

### Sibling predicate `IsMacroEntryTunnel` (same family, would move with it)

- `Decompiler/GraphWalker.cpp:191` — `ClassifyNode` → `ENodeSemantics::TunnelEntry`.
- `BlueprintHandlerUtils.cpp:2270`-ish — `FindMacroTunnelPair`, used for macro entry/exit
  pairing.

### Indirect consumers that inherit the blind spot today

Unchanged by a seed-only fix, but they are who benefits. Everything downstream of
`BuildExecReachabilitySet` / `FindBlueprintOrphanNodes`:

- `Handlers/Blueprint/BlueprintGraphOrphanHandler.cpp:217, 254, 310, 359` —
  `blueprint.graph.find_orphaned_nodes` and `blueprint.graph.delete_orphaned_nodes`.
- `Handlers/Blueprint/BlueprintEventHandler.cpp:539, 691` — `blueprint.remove_event`
  orphan-delta cleanup.
- `Handlers/Blueprint/BlueprintGraphCrudHandler.cpp:694, 710, 1094, 1815` — `delete_node`
  and node-replace cleanup.
- `Handlers/UI/WidgetHierarchyHandler.cpp:217, 231` — widget hierarchy mutation cleanup.
- `Decompiler/BpirDecompiler.cpp:910` — BPIR orphan warnings in `blueprint.decompile` and
  `asset.dump`.

### Tests pinned to the current definition

`Tests/Blueprint/TestExecutionFlowTimelineRoot.cpp:76, 83, 173` (asserts the EventGraph has
zero `IsBlueprintEntryNode` entries) · `Tests/Bpir/TestBlueprintGraphCompositeEntryPoints.cpp:207,
264, 357` (counterfactuals for the composite recursion) ·
`Tests/Bpir/TestBpirCallK2NodeDecompile.cpp:54` · `Tests/Bpir/TestDecompiler.cpp:764-805`
(`FindEntryPoints` returns 2) · `Tests/Bpir/TestBlueprintGraphOrphan.cpp:860` (knot-aware BFS
counterfactual) · `Tests/Bpir/TestBpirEmptyEntryConsistency.cpp:78` and
`Tests/Utility/TestBpirEmptyGraphMarker.cpp:137, 173` (both depend on `FindEntryPoints`
returning empty for a tunnel-less graph).

### Hard constraint on any fix

`B-bpir-entry-points-skip-composite-subgraphs` (DONE, Critical) exists **because** events
nested inside a collapsed graph were being missed, which falsely reported live composites as
orphans. A `K2Node_Event` / `K2Node_FunctionEntry` inside a bound graph is a real entry — the
engine registers its delegate regardless of how the outer node is wired — so any fix must keep
seeding those composites. Undoing that ticket to fix this one trades a Medium for a Critical.

## Fix

**A. Composite-liveness pass keyed on the outer node.** Delete the special case at `:2453` and
let a composite be an ordinary node in the exec/data walk — reachable iff something wires into
it — with one preserved exception: still seed it when its bound graph transitively contains a
*real* entry node (event / custom event / function entry / input event), i.e.
`IsBlueprintEntryNode` **minus the tunnel arm**. Correct semantics, honours the constraint
above.

**B. A tunnel-excluding predicate used only by the seed (recommended).** Same semantics as A,
expressed as the minimum delta: add `IsBlueprintRealEntryNode` (`IsBlueprintEntryNode` without
the `IsMacroEntryTunnel` arm) plus a `CollectRealEntryNodesRecursive`, and call **only** the new
pair from `BuildExecReachabilitySet:2456`. Every one of the nine sites above keeps the shared
predicate byte-for-byte; the blast radius collapses to one branch of one function. Two follow-on
details, both already satisfied: the delete path is safe because `UK2Node_Composite::DestroyNode`
removes the bound graph via `FBlueprintEditorUtils::RemoveGraph`, and stale bound-graph
descriptors killed by that teardown are already guarded by `CleanupNewBlueprintOrphans`'s
`WeakNodePtr.IsValid()` check.

**C. Accept and document.** Note in the `blueprint.graph.find_orphaned_nodes` wiki page that
composites are out of scope for the sweep and must be removed with `delete_node`. Zero risk,
leaves the dump-level false-negative in place.

**Take B.** It buys A's correctness for a one-branch change, keeps the shared entry-node
definition — the thing the Critical warned about — completely untouched, and is small enough to
be covered by a single new test. Ship C's wiki note alongside it either way; the sweep's scope
with respect to composites deserves to be written down.

Test to add: build a Blueprint with (a) a genuinely dead collapsed graph and (b) a live one, run
`find_orphaned_nodes`, assert exactly the dead composite is reported and the live one is not;
plus a third case with a `K2Node_Event` inside a dead-looking composite's bound graph, asserting
it is **not** reported (the `B-bpir-entry-points-skip-composite-subgraphs` guard).

Gated on `B-orphan-sweep-mathexpression-fatal-load` reaching DONE: that ticket's fix is filed
IN-REVIEW as not-compiled and not-run, it edits the same two functions, and making composites
reportable makes them deletable — the safety of deleting a composite is precisely what that
ticket's fix establishes.

## History
- `#1-composite-seed-is-tautology` `OPEN` reporter — Split out of `B-orphan-sweep-mathexpression-fatal-load` #4 as its own ticket. Verified at plugin HEAD `b4413416`: the composite seed arm in `BuildExecReachabilitySet` (`BlueprintHandlerUtils.cpp:2453`) tests `CollectEntryNodesRecursive(BoundGraph).Num() > 0`, which counts `IsMacroEntryTunnel` nodes; `UK2Node_Composite::PostPlacedNewNode` (`K2Node_Composite.cpp:310-338`) always creates an output-only entry tunnel, so the test is true for every well-formed composite and false only for already-corrupt ones. Result: a dead `UK2Node_Composite` / `UK2Node_MathExpression` is never an orphan in `find_orphaned_nodes`, `delete_orphaned_nodes`, the delta cleanups, or the BPIR orphan warnings, and it transitively anchors its whole upstream pure-data chain as reachable. Rated Medium and ergonomic rather than correctness: no data loss, compile prunes it for free at `KismetCompiler.cpp:3836` before expansion, cost is a per-load reconstruct plus dead subgraphs reading as live in dumps, and the workaround is one `delete_node` call on the composite guid. Nine direct consumers of `IsBlueprintEntryNode` / `CollectEntryNodesRecursive` enumerated in the body; changing the shared predicate would reintroduce the Critical at site #5 and blank every macro graph at sites #3 and #8, so recommend fix B — a tunnel-excluding predicate called only from the seed.
