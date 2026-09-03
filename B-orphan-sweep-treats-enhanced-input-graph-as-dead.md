---
id: B-orphan-sweep-treats-enhanced-input-graph-as-dead
title: "The reachability collector does not seed from K2Node_EnhancedInputAction, so an entire UE5 input graph reads as orphaned — find_orphaned_nodes reports 22 live nodes as dead and delete_orphaned_nodes would sweep every input binding in the Blueprint"
status: OPEN
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

## History
- `#1-filed` `OPEN` reporter — Found while auditing `/Game/FPS/Player/BP_FPSCharacter` (UE 5.8, EAContentExamples58) after authoring its ten Enhanced Input entry nodes. `find_orphaned_nodes` answered `orphanedCount: 26` on a 430-node Blueprint that compiles clean; 22 of the 26 were the ten `K2Node_EnhancedInputAction` nodes plus the twelve handler `CallFunction` nodes wired to their `Triggered`/`Started`/`Completed` pins. Disproved by the sibling verb: `get_execution_flow` with `startNodeId` set to the IA_Move node returns a two-node chain with the `Triggered -> HandleMove` exec link and the `ActionValue -> Value` data link both present. The give-away is in the response itself — the `entryPoints` array of both verbs lists only `K2Node_Event` nodes and omits all ten input nodes, so nothing downstream of them is reachable. Filed Critical because `delete_orphaned_nodes` is the paired write verb and would delete the entire input graph of any UE5 Blueprint, leaving it compiling and silently unresponsive. Related but distinct: `B-bpir-entry-points-skip-composite-subgraphs` (DONE) fixed *where* the same shared collector looks (inside composites) while this is *what class* it seeds from; `B-bpir-input-event-entry-signatures-unknown` (DONE) is the decompiler's naming of these same node classes and its list is a good checklist for the other classes likely affected here. Also observed in passing and worth its own look: `blueprint.remove_event` removed the `AddRecoil` event node (`removedNodeCount: 1`) but left its four body nodes behind as genuine orphans — those four were the only real orphans in the 26.
