---
id: B-orphan-sweep-deletes-isolated-timeline
title: "An isolated K2Node_Timeline is reported as an orphan and deleted by delete_orphaned_nodes, which also destroys its UTimelineTemplate — the engine roots Timelines by type and never prunes them"
status: IN-REVIEW
severity: High
category: bug
tags: [orphan-detection, find-orphaned-nodes, delete-orphaned-nodes, reachability, entry-points, k2node-timeline, auto-play, timeline-template, data-loss, false-positive, decompiler, bpir]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# An isolated Timeline chain is swept as dead, and the sweep destroys the timeline template

Split out of `B-orphan-sweep-treats-enhanced-input-graph-as-dead` (#2), which
replaced the hard-coded entry seed with a port of the engine's compile root set but
deliberately omitted the engine's `UK2Node_Timeline` clause. This ticket is the
consequence of that omission. Filed separately because it needs a different fix in a
different function, and because the stated reason for the omission does not hold.

## Verified mechanism (plugin `2a671efb`, UE 5.8, source reading only)

`IsEngineCompileRootSetNode` (`BlueprintHandlerUtils.cpp:2238`) ports
`UE::KismetCompiler::Private::GatherRootSet` but keeps only two of its three clauses:
`UK2Node::IsNodeRootSet()` and *impure `UK2Node` with zero input pins*. The engine's
third clause is a type test:

```cpp
// KismetCompiler.cpp:114
const bool bRootSetByType = Node && (Node->IsA<UK2Node_FunctionEntry>()
    || Node->IsA<UK2Node_Event>() || Node->IsA<UK2Node_Timeline>());
```

Neither retained clause can ever reach a Timeline:

- `UK2Node_Timeline` does not override `IsNodeRootSet()`, so it inherits
  `K2Node.h:315`'s `return false`.
- `UK2Node_Timeline::AllocateDefaultPins` (`K2Node_Timeline.cpp:121-137`) creates
  **six** input pins before anything else — `Play`, `PlayFromStart`, `Stop`,
  `Reverse`, `ReverseFromEnd`, `SetNewTime`, plus the `NewTime` float — so the
  zero-input-pin shape clause returns false at the first pin.

`IsBlueprintEntryNode` (`:2269`) therefore rejects it, and that predicate is the sole
seed of `BuildExecReachabilitySet` (`:2489`, seed loop at `:2518-2534`). A Timeline
whose `Play`/`PlayFromStart`/`Reverse` inputs are unwired is reachable from nothing, so
it and every node hanging off its `Update`/`Finished`/event-track outputs land in
`ScanGraphForBlueprintOrphans`'s report (`:2656`, entry skip at `:2677`).
`bAutoPlay` is irrelevant on both sides: the engine roots by type unconditionally, and
PinWright reads the flag nowhere in this path.

## Why the divergence rationale in the parent ticket does not hold

The parent's `#2` argued the case is already covered by `CollectLatentExecRootNodes`.
It is not, for two independent reasons:

1. **Wrong consumer.** `CollectLatentExecRootNodes` (`:2461`) has exactly one non-test
   caller: `get_execution_flow` (`BlueprintGraphInspectionHandler.cpp:1383`). The
   orphan sweep never calls it. `BuildExecReachabilitySet` has two consumers — the
   sweep and the BPIR decompiler's orphan-warning pass (`BpirDecompiler.cpp:965`) —
   and neither sees latent roots.
2. **Wrong gate even there.** The fallback is guarded by
   `if (EntryNodes.Num() == 0)` (`BlueprintGraphInspectionHandler.cpp:1381`). An
   auto-play Timeline sitting in an EventGraph that *also* has a `BeginPlay` never
   reaches it. That mixed shape is not hypothetical: it is the exact repro in the
   IN-REVIEW ticket `B-decompile-orphan-pure-nodes-grafted` ("an auto-play Timeline
   whose `Play` pin is never wired from any event ... **alongside** at least one
   reachable entry point").

## Blast radius: this deletes more than a graph node

`delete_orphaned_nodes` calls `FBlueprintEditorUtils::RemoveNode`
(`BlueprintGraphOrphanHandler.cpp:153`), which calls `Node->DestroyNode()`
(`BlueprintEditorUtils.cpp:2868`, under the engine's own comment "Timelines will be
removed from the blueprint if the node is a UK2Node_Timeline").
`UK2Node_Timeline::DestroyNode` (`K2Node_Timeline.cpp:204-218`) then calls
`FBlueprintEditorUtils::RemoveTimeline` (`BlueprintEditorUtils.cpp:8143-8156`), which
drops the template from `Blueprint->Timelines` and `MarkAsGarbage()`es it, and renames
it into the transient package.

So the sweep destroys the `UTimelineTemplate` — every float/vector/linear-color/event
track and its curves, `TimelineLength`, `bLoop`, `bAutoPlay`, `bReplicated`, and the
timeline variable itself — not just the node. The handler then compiles and **saves**
(`BlueprintGraphOrphanHandler.cpp:171-172`), so the loss is written to disk in the same
call. Re-adding a Timeline node afterwards recreates an empty template; the authored
curves are gone. This is unrecoverable without VCS, and there is no
`COMPOSITE_BOUNDARY_BROKEN`-style guard for it.

## Reproduction shape

Any Blueprint graph containing a `K2Node_Timeline` whose exec **inputs** are all
unwired while its exec **outputs** drive a chain. Two field-observed instances of that
shape are already on this board:

- `E-execflow-no-event-entry-timeline-root` — `/Game/ExampleContent/Blueprints/
  Blueprints/BP_Timeline_Ball`, a shipped Epic Content Examples asset whose EventGraph
  has zero `K2Node_Event`/`K2Node_FunctionEntry` nodes and whose only root is the
  auto-play Timeline "Bounce" (its own comment: "This timeline is set to play and loop
  automatically"). `find_orphaned_nodes` on that graph reports the Timeline plus all
  four downstream nodes; `delete_orphaned_nodes` empties the graph and takes the
  bounce curve with it.
- `B-decompile-orphan-pure-nodes-grafted` — the same auto-play Timeline shape
  *alongside* a live entry, which is the variant the `get_execution_flow` fallback
  cannot rescue.

Sweep of this repo's committed dump mirror (`asset-dumps/`, `grep Timeline **/bpir.txt`):
12 Blueprints contain Timelines; the 10 with a decompilable body all wire `Play`/
`PlayFromStart`/`SetNewTime` from an event, so none is currently at risk here. The two
Timelines flagged as orphans in that mirror
(`Game/ScifiJungle/ExamplePlayer/Blueprints/BP_ExamplePlayer`, `EventGraph ::
'Timeline_0'` and `CameraHandling :: 'Timeline'`) sit in warning clusters whose node
coordinates line up with `K2Node_EnhancedInputAction` chains at the same Y band, so
they are almost certainly collateral of the parent ticket's bug and should clear once
that fix lands — the mirror predates it. **Not confirmed**: attributing those two
would need a live `get_execution_flow`, which this report did not run.

## Expected

The sweep's reachability seed matches the engine's root set for the nodes the engine
roots. A node the Kismet compiler refuses to prune is not an orphan, and the shipped
runtime proves it: an auto-play timeline actually ticks.

## Fix

**Do not** add `UK2Node_Timeline` to `IsBlueprintEntryNode` / `IsEngineCompileRootSetNode`.
That predicate is shared with `CollectEntryNodesRecursive`, `get_execution_flow`'s
`entryPoints`, and the BPIR decompiler's entry selection, and promoting Timelines there
causes three regressions:

1. `FBpirTextEmitter::EmitEntrySignature` (`BpirTextEmitter.cpp:1186`) has no
   `UK2Node_Timeline` arm, so every Timeline would render as
   `entry event UnknownEntry()` and be hoisted out of its caller's block — breaking the
   currently-correct `%n1: enum<ETimelineDirection> = timeline Move() [update -> @update]`
   rendering of all 10 wired Timelines in this repo's dump mirror, and breaking the BPIR
   round-trip.
2. `CollectLatentExecRootNodes` skips `IsBlueprintEntryNode` nodes (`:2470`), so the
   latent-root path would go dead and `entryPointsAreLatentRoots` would flip to false.
3. `TestExecutionFlowTimelineRoot.cpp:174`, `:179-181` and `:230` all fail.

The correct change is scoped to the sweep. Seed `BuildExecReachabilitySet`
(`BlueprintHandlerUtils.cpp:2518-2534`) from `IsBlueprintEntryNode(Node) ||
Node->IsA<UK2Node_Timeline>()` — a sweep-local root predicate, e.g.
`IsOrphanSweepRootNode`, so the intent is documented at the seed rather than smuggled
into a shared entry test. That single edit covers both consumers of the reachability
set (the sweep and the BPIR orphan warnings), leaves `entryPoints`, the latent-root
fallback and `EmitEntrySignature` untouched. The existing timeline entry-point tests
remain unchanged; the mixed-graph sweep regression is added below.
`UK2Node_Timeline` lives in the always-linked `BlueprintGraph` module and is already
included in six plugin translation units, so no optional-module guard is needed.

Note the Timeline stays *reported* as a non-entry everywhere else, which is what the
existing timeline tests pin — the sweep just stops treating it as unreachable.

Implementation: `BlueprintHandlerUtils.cpp` now includes `K2Node_Timeline.h` and uses the
sweep-local `IsOrphanSweepRootNode` predicate at the `BuildExecReachabilitySet` seed. The
structural automation test `PinWright.blueprint.graph.find_orphaned_nodes.TimelineWithLiveEntry`
reuses `TestExecutionFlowTimelineRoot.cpp`'s fixture, adds a live `BeginPlay` chain and a dead
`Branch`, and asserts that only the dead Branch is reported. `IsBlueprintEntryNode`, the
latent-root collector, `BpirTextEmitter.cpp`, and `BlueprintGraphOrphanHandler.cpp` are
deliberately unchanged; no live PIE, build, or Unreal test run was performed.

Regression test to add alongside: an auto-play-shaped Timeline (unwired exec inputs,
`Update` wired to a `PrintString`) in a graph that ALSO has a live `BeginPlay` chain
plus one genuinely dead `Branch`; assert `orphanedCount == 1` and that neither the
Timeline nor its downstream node is listed. The existing
`TestExecutionFlowTimelineRoot.cpp` fixture builder is reusable for the Timeline half.

**Workaround:** never run `delete_orphaned_nodes` on a Blueprint containing a Timeline.
Re-check any reported `K2Node_Timeline` with `get_execution_flow` using that node's id
as `startNodeId`, and delete confirmed-dead nodes one at a time with
`blueprint.graph.delete_node`.

severity rationale: impact=Critical (a write that loses asset data — the sweep deletes
a live node *and* garbage-collects its UTimelineTemplate curves, then saves) x
reach=narrower than the parent (only Blueprints holding a Timeline with unwired exec
inputs; 0 of the 12 Timeline-bearing Blueprints in this repo's dump mirror qualify,
though two field instances are already on this board) -> one reach step down -> High.
The read-only half compounds it independently at High impact: `find_orphaned_nodes`
and `blueprint.decompile` report a live, running auto-play chain as dead on every call.

## History
- `#1-split-from-enhanced-input-fix` `OPEN` reporter — Split out of `B-orphan-sweep-treats-enhanced-input-graph-as-dead` (#2), whose fix ported `UE::KismetCompiler::Private::GatherRootSet` but dropped its `UK2Node_Timeline` type clause. Confirmed by source reading at plugin `2a671efb` against UE 5.8: `UK2Node_Timeline` inherits `IsNodeRootSet() == false` (`K2Node.h:315`) and allocates six exec input pins in `AllocateDefaultPins` (`K2Node_Timeline.cpp:121-137`), so neither retained clause of `IsEngineCompileRootSetNode` (`BlueprintHandlerUtils.cpp:2238`) can reach it, while the engine roots it unconditionally by type (`KismetCompiler.cpp:114`) in the pre-expansion `PruneIsolatedNodes` pass (`:2031-2039`, `:3833-3835`). The parent's rationale — "PinWright resolves auto-play timelines via CollectLatentExecRootNodes" — fails twice: that collector's only non-test caller is `get_execution_flow` (`BlueprintGraphInspectionHandler.cpp:1383`), never the sweep, and it is gated on `EntryNodes.Num() == 0` (`:1381`) so it does not fire when the Timeline shares a graph with a live event — the exact shape in the IN-REVIEW ticket `B-decompile-orphan-pure-nodes-grafted`. Escalated above a plain false positive because `DestroyNode` on a Timeline node runs `FBlueprintEditorUtils::RemoveTimeline` (`BlueprintEditorUtils.cpp:8143-8156`), which `MarkAsGarbage()`es the `UTimelineTemplate` — curves, tracks, length, loop/autoplay flags — and the handler compiles and saves immediately after (`BlueprintGraphOrphanHandler.cpp:171-172`). Rejected the obvious fix (root by type in `IsBlueprintEntryNode`) because that predicate also drives `entryPoints`, the latent-root fallback, and BPIR entry selection, and `EmitEntrySignature` (`BpirTextEmitter.cpp:1186`) has no Timeline arm — it would regress all 10 correctly-rendered wired Timelines in this repo's dump mirror to `entry event UnknownEntry()` and fail three assertions in `TestExecutionFlowTimelineRoot.cpp`. Proposed instead a sweep-local seed in `BuildExecReachabilitySet` (`:2518-2534`), which fixes both consumers of the reachability set and needs no test changes. Dump-mirror sweep found 12 Timeline-bearing Blueprints and no confirmed isolated one in this checkout; the two Timelines flagged as orphans in `BP_ExamplePlayer` are coordinate-aligned with EnhancedInput chains and are most likely parent-ticket collateral in a pre-fix dump — not verified live.
- `#2-timeline-sweep-root` `IN-REVIEW` developer — TRUE confirmed from `BlueprintHandlerUtils.cpp`: the orphan reachability seed now uses the sweep-local `IsOrphanSweepRootNode` (`IsBlueprintEntryNode(Node) || Node->IsA<UK2Node_Timeline>()`), so Timeline output chains remain reachable even when a graph also has a live entry. Added `Source/PinWright/Private/Tests/Blueprint/TestExecutionFlowTimelineRoot.cpp` automation test `PinWright.blueprint.graph.find_orphaned_nodes.TimelineWithLiveEntry`, which expects one planted dead Branch and excludes the Timeline plus both live PrintString chains. Deliberate non-changes: `IsBlueprintEntryNode`, `CollectLatentExecRootNodes`, `BpirTextEmitter.cpp:1186`, and `BlueprintGraphOrphanHandler.cpp` were not modified; no build, live PIE, or Unreal test execution was performed.
