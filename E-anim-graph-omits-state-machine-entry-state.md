---
id: E-anim-graph-omits-state-machine-entry-state
title: "`get_animation_info` / `anim_graph.json` `state_machines[]` omits the entry state — `set_state_machine_entry`'s effect is unverifiable in the structured readback (changing the entry leaves the dump byte-identical)"
status: OPEN
severity: Medium
category: ergonomic
tags: [animation, anim-graph, state-machine, entry-state, readback, get-animation-info, asset-dump, anim-graph-json, set-state-machine-entry]
encounters: 2
lastSeen: 2026-06-25T07:14:35Z
---

# The structured state-machine readback never reports which state is the entry

`animation.authoring.set_state_machine_entry` "Rewires a state machine's
entry node to the named state" — and the `add_state_machine` wiki overlay
documents the entry as a first-class, meaningful property ("the entry state
is implicit — the first state added with `add_state` becomes the default";
`F-anim-state-machine-internals` `#1` lists "Editing an existing state
machine's entry state" as a real use case). The entry node is what decides
which state the machine starts in.

But the structured readback emits no entry-state field at all. The
`state_machines[]` object built by `AnimGraphDumpBuilder::BuildAnimGraphJson`
(`Source/PinWright/Private/Handlers/Animation/AnimGraphDumpBuilder.cpp:182-264`)
sets only `name`, `page`, `states[]`, `transitions[]`, and `conduits[]`. The
loop walks `MachineGraph->Nodes` for `UAnimStateNode` / `UAnimStateConduitNode`
/ `UAnimStateTransitionNode` but never reads
`UAnimStateMachineGraph::EntryNode` (or follows the entry node's output exec to
its target state). This builder is the same code `get_animation_info` now merges
for an AnimBlueprint (via `MergeMissingFields`) and that `asset.dump` writes to
`anim_graph.json`, so **both** structured readbacks omit the entry.

Net ergonomic harm: after `set_state_machine_entry` succeeds, a caller reading
back via `get_animation_info` (the documented inspect-after-mutate dual) cannot
confirm — or even observe — which state is the entry. Changing the entry leaves
the readback byte-identical (verified below), so the only confirmation of an
entry rewire is the write call's own `{success:true}`; the canonical readback is
blind to it. The states list also has no positional/ordering guarantee that
would let a consumer infer the implicit default, so even the
"first-added-state-is-entry" convention isn't recoverable from the dump.

This is the same readback-omits-an-authored-field class as
`E-anim-graph-json-omits-transition-logic-blend` (IN-REVIEW, Medium —
`set_transition_settings`'s `logicType`/`blendMode` missing from
`transitions[]`); here it's `set_state_machine_entry`'s output missing from the
`state_machines[]` object. It is distinct from:
- `F-anim-state-machine-internals` (DONE) — that added the **write** verb
  `set_state_machine_entry`; the readback was never extended to surface it.
- `E-get-animation-info-thin-on-anim-blueprint` (OPEN) — that was the AnimBP
  branch emitting only metadata; the topology now merges through, yet even with
  the full topology present the entry state is still absent (an independent
  field gap in the builder, not a missing-merge gap).
- `E-state-machine-subgraph-name-default` (OPEN) — that's the sub-graph `name`
  carrying the engine default; orthogonal to the missing entry field.

## Workaround
Trust `set_state_machine_entry`'s `{success:true}` response (it does not echo
the resulting entry either, but it does report success); or, for the unmodified
implicit default, assume the first `add_state` is the entry. There is no
readback path that reports the actual current entry state.

## Fix
Add an entry-state field to the `state_machines[]` object in
`AnimGraphDumpBuilder::BuildAnimGraphJson` (e.g. `entry_state`, the
`GetStateName()` of the state the `UAnimStateMachineGraph::EntryNode`'s output
exec links to; empty string when unset). `EntryNode` is public on
`UAnimStateMachineGraph`; resolve its linked-to state via the entry node's
output pin → `UAnimStateNodeBase`. Because this changes serialized
`anim_graph.json` bytes, bump that aspect's version in
`AssetDumpCache.cpp` (`Versions` table) in the same commit so stale dumpcache
entries regenerate. `get_animation_info` picks the field up for free (it merges
this builder). For symmetry, `set_state_machine_entry`'s success response could
also echo the resulting `entryState`.

## Verbatim repro

Built ABP `ABP_MannequinLocomotion` (skeleton
`/Game/Characters/Mannequin_UE4/Meshes/SK_Mannequin_Skeleton`) with state
machine `Locomotion`, states `Idle`/`Run`, transition `Idle->Run`. Then:

1. `animation.authoring.get_animation_info`
   `{assetPath:"/Game/Animations/ABP_MannequinLocomotion"}` — the state machine
   object carries no entry field:

   ```json
   "state_machines":[{"name":"Locomotion","page":"AnimGraph",
     "states":[{"name":"Idle","guid":"8242d615-..."},{"name":"Run","guid":"657fd069-..."}],
     "transitions":[{"from":"Idle","to":"Run","priority":1,"rule_graph":"Transition",
       "bidirectional":false,"disabled":false,"logic_type":"StandardBlend","blend_mode":"Linear"}],
     "conduits":[]}]
   ```

2. `animation.authoring.set_state_machine_entry`
   `{blueprintPath:"/Game/Animations/ABP_MannequinLocomotion", stateMachineName:"Locomotion", stateName:"Run"}`
   → `{"stateMachine":"Locomotion","stateName":"Run","success":true}` (entry
   rewired Idle → Run).

3. `animation.authoring.get_animation_info` (same args as step 1) → **byte-identical**
   to step 1's output. The entry change from Idle to Run is invisible; the
   readback cannot distinguish entry=Idle from entry=Run.

Source-confirmed: `AnimGraphDumpBuilder.cpp:182-264` emits only
`name`/`page`/`states`/`transitions`/`conduits` and never reads
`EntryNode`.

## History
- `#2-additional-agir-and-priority-default` `OPEN` reporter — Additional evidence (SEED-mode, seed `animation.authoring.add_state_machine`): re-confirmed on a fuller topology — ABP `ABP_OracleReplaySM` (skeleton `SK_Mannequin_Skeleton`), state machine `Locomotion`, states `Idle`/`Walk`/`Run`, transitions `Idle->Walk`/`Walk->Run`/`Run->Walk`/`Walk->Idle`, then `set_state_machine_entry stateName=Idle`. `get_animation_info` and `asset.dump`'s `anim_graph.json` `state_machines[].{name,page,states,transitions,conduits}` carry no entry field, matching `#1`. Additionally: the **text** readback `agir.txt` is also blind to the entry — `FAGIRTextEmitter::EmitGraph` (`Source/PinWright/Private/AGIR/AGIRTextEmitter.cpp`) emits the `state_machine`/`state`/`transition` lines but no entry-node line, so NO readback surface (structured JSON, dump JSON, or text AGIR) reports the entry. Side note while replaying (NOT a bug): all four transitions read back `priority:1` though only `Idle->Walk` was given `priorityOrder:1` — this is faithful, the engine default `UAnimStateTransitionNode::PriorityOrder = 1` (`Engine/Source/Editor/AnimGraph/Private/AnimStateTransitionNode.cpp:96`), so the `set_transition_rules priorityOrder:1` call was a no-op-equal-to-default, not a misreport. The fix should ideally surface `entry_state` in `agir.txt` too, not only the JSON.
- `#1-initial-repro` `OPEN` reporter — SEED-mode locomotion AnimBP build-out (seed `animation.create_anim_blueprint`). Attempt succeeded; the friction note flagged that `get_animation_info` "exposes no explicit entry state field, so the Idle entry is confirmed only via the set_state_machine_entry success result." Replay-confirmed: `set_state_machine_entry` rewired the entry Idle→Run (`{success:true}`), and a subsequent `get_animation_info` returned output byte-identical to the pre-change readback — the entry state is unobservable in the structured dump. Source-confirmed `AnimGraphDumpBuilder::BuildAnimGraphJson` (`Source/PinWright/Private/Handlers/Animation/AnimGraphDumpBuilder.cpp:182-264`) sets only `name`/`page`/`states`/`transitions`/`conduits` and never reads `UAnimStateMachineGraph::EntryNode`; the same builder backs both `get_animation_info` (merged for AnimBP) and `asset.dump`'s `anim_graph.json`. Deduped: distinct from `F-anim-state-machine-internals` (DONE — added the write verb only), `E-get-animation-info-thin-on-anim-blueprint` (OPEN — topology-merge gap, now resolved but entry still absent), `E-anim-graph-json-omits-transition-logic-blend` (IN-REVIEW — same readback-omits-authored-field class, transition fields), and `E-state-machine-subgraph-name-default` (OPEN — sub-graph naming). Fix: emit `entry_state` (the `EntryNode`-linked state's `GetStateName()`) in the `state_machines[]` object and bump the `anim_graph.json` aspect version in `AssetDumpCache.cpp`.
