---
id: B-orphan-sweep-treats-enhanced-input-graph-as-dead
title: "The reachability collector does not seed from K2Node_EnhancedInputAction, so an entire UE5 input graph reads as orphaned — find_orphaned_nodes reports 22 live nodes as dead and delete_orphaned_nodes would sweep every input binding in the Blueprint"
status: IN-REVIEW
severity: Critical
category: bug
tags: [orphan-detection, find-orphaned-nodes, delete-orphaned-nodes, reachability, entry-points, enhanced-input, K2Node_EnhancedInputAction, false-positive, execution-flow, data-loss]
---

# Enhanced Input event nodes are not entry points to the reachability collector

## Symptom

On a Blueprint-only UE5 first-person character with ten `UK2Node_EnhancedInputAction`
nodes (the only way a Blueprint receives Enhanced Input), `find_orphaned_nodes` reports
**26 orphans out of 430 nodes — 22 of them live and correctly wired**:

```
call("blueprint.graph.find_orphaned_nodes", {assetPath: "/Game/FPS/Player/BP_FPSCharacter"})
-> orphanedCount: 26
   ... all 10 K2Node_EnhancedInputAction nodes (IA_Move, IA_Look, IA_Fire, IA_ADS,
       IA_Reload, IA_Sprint, IA_Crouch, IA_Jump, IA_Melee, IA_Pause)
   ... plus the 12 K2Node_CallFunction handler calls wired to their exec pins
       (HandleMove, HandleLook, HandleADSStart/Stop, HandleSprintStart/Stop,
        HandleCrouchStart/Stop, HandleJumpStart/Stop, HandleReload, HandleMelee)
```

The same call's own `entryPoints` list — and `get_execution_flow`'s — contains only
`K2Node_Event` nodes (`Event Tick`, `Event BeginPlay`, `Event AddRecoil`,
`Event PointDamage`, `Event ActorBeginOverlap`). The ten input nodes are absent, so
nothing downstream of them is reachable and the whole input graph is classified dead.

## Proof the nodes are live

`get_execution_flow` started **at** one of the "orphans" walks a correct chain:

```
call("blueprint.graph.get_execution_flow", {assetPath: "...", graphName: "EventGraph",
     startNodeId: "<IA_Move node>", maxDepth: 3, compact: true})
-> executionChain:
   [0] K2Node_EnhancedInputAction "EnhancedInputAction IA_Move"
       execOutputs: [{pin: "Triggered", targetNodeId: "<HandleMove call>"}]
   [1] K2Node_CallFunction "HandleMove"
       dataInputs: [{pin: "Value", source: "<IA_Move node>"}]
```

Exec link present, `ActionValue` data link present, the Blueprint compiles clean. The
nodes are not orphans by any definition except the collector's.

## Why this is Critical rather than cosmetic

`delete_orphaned_nodes` is the paired write verb, and its wiki text
(`blueprint.graph`) presents the sweep as a cleanup. Running it on any UE5 Blueprint
that takes player input **deletes every input binding in it** — the ten entry nodes
and the handler calls hanging off them — and the Blueprint still compiles afterwards,
so the loss is silent until someone plays the game and nothing responds to the
keyboard. There is no warning on either verb that input nodes are outside its
reachability model.

It also poisons an honest workflow: an agent that (correctly) uses
`find_orphaned_nodes` to check its own graph for leftovers cannot distinguish its four
real orphans from the twenty-two false ones without hand-walking every reported node
through `get_execution_flow`. In this session there were exactly four genuine orphans
(a dead `AddRecoil` body left behind by `blueprint.remove_event`), buried in 22 false
positives.

## Root cause guess

`B-bpir-entry-points-skip-composite-subgraphs` (DONE) describes the shared collector as
walking `K2Node_Event` and names the same four consumers — `blueprint.decompile`,
`blueprint.inspect`, `blueprint.graph.get_execution_flow`,
`blueprint.graph.find_orphaned_nodes`. `UK2Node_EnhancedInputAction` does **not**
derive from `UK2Node_Event` (it is a `UK2Node` that expands into bindings at compile
time), so a `K2Node_Event`-typed seed misses it entirely. That earlier ticket fixed
*where* the collector looks (nested composites); this one is about *what class* it
seeds from. No plugin source was read for this report — the class relationship is from
the engine, the collector behaviour is from the responses above.

The same blind spot should be checked for the other non-`K2Node_Event` entry classes
the DONE ticket `B-bpir-input-event-entry-signatures-unknown` enumerated:
`UK2Node_InputAxisEvent`, `UK2Node_InputKeyEvent`, `UK2Node_InputTouchEvent`,
`UK2Node_InputVectorAxisEvent`, `UK2Node_ActorBoundEvent`,
`UK2Node_GeneratedBoundEvent`, `UK2Node_WidgetAnimationEvent`.

## Expected

- The reachability collector seeds from every entry-shaped K2Node, not only
  `K2Node_Event`. `UK2Node::IsNodeRootSet()` / the presence of an entry-style exec
  output with no exec input is a better test than a class check.
- Until that lands, `delete_orphaned_nodes` should refuse (or at minimum warn loudly)
  when the sweep set contains an input-event node, because that is never a real
  orphan.

## Workaround used

Treat `find_orphaned_nodes` output as a candidate list only: re-check every reported
node with `get_execution_flow` from that node id, and delete confirmed dead ones
one at a time with `blueprint.graph.delete_node`. Never run
`delete_orphaned_nodes` on a Blueprint that handles input.

severity rationale: impact=silent data loss (the paired write verb deletes a working
input graph and the Blueprint still compiles) x reach=every UE5 Blueprint that takes
player input -> Critical

## Fix

Confirmed true by source reading. `BlueprintHandlerUtils::IsBlueprintEntryNode`
(`Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp`) seeded from a
hard-coded `IsA<>` list of entry classes; `UK2Node_EnhancedInputAction` derives from
`UK2Node`, not `UK2Node_Event`, so it matched nothing. That one predicate is the seed for
`BuildExecReachabilitySet`, the skip in `ScanGraphForBlueprintOrphans`, and
`CollectEntryNodesRecursive` (which feeds `get_execution_flow`'s `entryPoints` and the BPIR
decompiler) — hence all four symptoms from one cause.

Fix is engine-aligned and class-agnostic, not another seed-list entry. The predicate now
falls through to `IsEngineCompileRootSetNode`, a port of the engine's own compile root set
(`UE::KismetCompiler::Private::GatherRootSet`, `KismetCompiler.cpp:110-141`, as reached from
`FKismetCompilerContext::ExpansionStep` with `bIncludeNodesThatCouldBeExpandedToRootSet=true`
— the pre-expansion pass, over the same authored graph this sweep walks):
`UK2Node::IsNodeRootSet()`, or **impure `UK2Node` with no input pins at all**. The second
clause is a shape test, so every event-shaped class is covered without naming it or linking
the EnhancedInput plugin's optional `InputBlueprintNodes` module (the plugin ships for UE
5.3-5.8). An impure node that *has* input pins — stranded `PrintString`, unwired `Branch` —
still reports, so real orphans are unaffected.

Deliberate divergence from the engine clause: `UK2Node_Timeline` is root-set by type there
but not added here. It has input pins so the shape clause never reaches it, PinWright already
resolves auto-play timelines through `CollectLatentExecRootNodes`, and
`TestExecutionFlowTimelineRoot.cpp:173` asserts the opposite. An isolated Timeline chain is
therefore still sweepable — worth its own ticket, not folded into this one.

No `delete_orphaned_nodes` refusal guard was added: with the seed corrected the input-node
false positives no longer exist, and a class-name guard would be the same antipattern.

Files changed:
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp` — added static
  `IsEngineCompileRootSetNode`, called as the general backstop at the end of
  `IsBlueprintEntryNode`; added `#include "K2Node.h"`.
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.h` — doc comment on
  `IsBlueprintEntryNode` restates the definition as the engine root set.
- `Source/PinWright/Private/Tests/Bpir/TestBlueprintGraphOrphanRootSet.cpp` — new, 2 tests.
- `docs/wiki-src/blueprint.graph.md` — new "What counts as an entry point" section.

Verification a reviewer should run:
1. `PinWright.blueprint.graph.find_orphaned_nodes.EnhancedInputEntryNotOrphaned` — builds a
   real `K2Node_EnhancedInputAction` (resolved by class path, skipped with
   `PINWRIGHT_ASSERTIONS_SKIPPED reason=fixture-unavailable` if EnhancedInput is off), wires
   its `Triggered` pin to a handler, plants one genuinely dead `Branch`, and asserts
   `orphanedCount == 1` with the input node and handler absent from the list.
2. `PinWright.blueprint.graph.find_orphaned_nodes.EntryRootSetPredicate` — asserts the node is
   not a `UK2Node_Event`, that `IsBlueprintEntryNode` accepts it, that an unwired `Branch` is
   *not* promoted, and that `CollectEntryNodesRecursive` lists it.
3. Whole existing orphan suite (`PinWright.bpir.graph_orphan.*`,
   `PinWright.blueprint.graph.*orphaned_nodes.*`) plus
   `PinWright.blueprint.get_execution_flow.*` — the timeline-root test's
   "EventGraph has no IsBlueprintEntryNode entries" precondition must still hold.
4. Live: `find_orphaned_nodes` on `/Game/FPS/Player/BP_FPSCharacter` should now report the
   4 real orphans (the dead `AddRecoil` body), not 26.

Known side effect, out of scope: `blueprint.decompile` now renders these nodes as entries and
`FBpirTextEmitter::EmitEntrySignature` has no arm for the class, so they emit
`entry event UnknownEntry()` (previously the whole input subgraph was dropped from the output
with an orphan warning). Naming belongs to `B-bpir-input-event-entry-signatures-unknown`.

## History
- `#1-filed` `OPEN` reporter — Found while auditing `/Game/FPS/Player/BP_FPSCharacter` (UE 5.8, EAContentExamples58) after authoring its ten Enhanced Input entry nodes. `find_orphaned_nodes` answered `orphanedCount: 26` on a 430-node Blueprint that compiles clean; 22 of the 26 were the ten `K2Node_EnhancedInputAction` nodes plus the twelve handler `CallFunction` nodes wired to their `Triggered`/`Started`/`Completed` pins. Disproved by the sibling verb: `get_execution_flow` with `startNodeId` set to the IA_Move node returns a two-node chain with the `Triggered -> HandleMove` exec link and the `ActionValue -> Value` data link both present. The give-away is in the response itself — the `entryPoints` array of both verbs lists only `K2Node_Event` nodes and omits all ten input nodes, so nothing downstream of them is reachable. Filed Critical because `delete_orphaned_nodes` is the paired write verb and would delete the entire input graph of any UE5 Blueprint, leaving it compiling and silently unresponsive. Related but distinct: `B-bpir-entry-points-skip-composite-subgraphs` (DONE) fixed *where* the same shared collector looks (inside composites) while this is *what class* it seeds from; `B-bpir-input-event-entry-signatures-unknown` (DONE) is the decompiler's naming of these same node classes and its list is a good checklist for the other classes likely affected here. Also observed in passing and worth its own look: `blueprint.remove_event` removed the `AddRecoil` event node (`removedNodeCount: 1`) but left its four body nodes behind as genuine orphans — those four were the only real orphans in the 26.

- `#2-root-set-seed-fix` `IN-REVIEW` developer — Verified true from source: `IsBlueprintEntryNode` (`BlueprintHandlerUtils.cpp`) was a hard-coded `IsA<>` class list, and it is the single seed for `BuildExecReachabilitySet`, the skip in `ScanGraphForBlueprintOrphans`, and `CollectEntryNodesRecursive` — so one missing class killed the sweep, `entryPoints`, and the decompiler at once. Replaced the "extend the list" fix with the engine's own predicate: added `IsEngineCompileRootSetNode`, a port of `UE::KismetCompiler::Private::GatherRootSet` (`KismetCompiler.cpp:110-141`) as invoked pre-expansion with `bIncludeNodesThatCouldBeExpandedToRootSet=true`, whose operative clause is a SHAPE test — impure `UK2Node` with zero input pins — plus `IsNodeRootSet()`. `UK2Node_EnhancedInputAction` allocates output-only pins (`K2Node_EnhancedInputAction.cpp AllocateDefaultPins`), so it is covered without naming the class or linking the optional `InputBlueprintNodes` module, and so is every future node of that shape. Impure nodes that have input pins still report, so genuine orphans are untouched. `UK2Node_Timeline` (root-set by type in the engine) is deliberately NOT added — it has input pins, PinWright routes auto-play timelines through `CollectLatentExecRootNodes`, and `TestExecutionFlowTimelineRoot.cpp:173` pins the opposite; an isolated Timeline chain staying sweepable is a separate ticket. No `delete_orphaned_nodes` refusal guard: the false positives are gone at the source. Two new tests in `Tests/Bpir/TestBlueprintGraphOrphanRootSet.cpp` build a real Enhanced Input node by class path (skip-marked when the plugin is off) and assert both the handler result and the predicate itself. Not compiled or run by this agent — a separate compile pass follows. Side effect left open: decompile now emits `entry event UnknownEntry()` for these nodes (previously it dropped the subgraph entirely), which is `B-bpir-input-event-entry-signatures-unknown`'s territory.
- `#N-verified-in-fps-build` `IN-REVIEW` reporter — **Fixed, verified on the exact graph that produced the original report.** UE 5.8 / EAContentExamples58, PinWright rebuilt from `origin/master` (waves 4-9), wiki regenerated. Call: `blueprint.graph.find_orphaned_nodes {assetPath: "/Game/FPS/Player/BP_FPSCharacter", graphName: "EventGraph", includeDataOnly: true}` -> `{"totalNodes": 97, "orphanedNodes": [], "orphanedCount": 0}`. The same graph previously reported **22** orphans — ten `K2Node_EnhancedInputAction` nodes (`IA_Move`, `IA_Look`, `IA_Fire`, `IA_ADS`, `IA_Reload`, `IA_Sprint`, `IA_Crouch`, `IA_Jump`, `IA_Melee`, `IA_Pause`) plus their twelve `Handle*` call nodes, every one of them live and reachable at runtime. `delete_orphaned_nodes` on that answer would have deleted the entire input graph. The graph is unchanged since then apart from additions, and `blueprint.decompile` still emits all ten input entries, so `orphanedCount: 0` is the correct answer rather than an empty scan: `totalNodes: 97` confirms it walked the whole graph. Left `IN-REVIEW` per the resume protocol.
