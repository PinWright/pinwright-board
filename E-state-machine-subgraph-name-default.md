---
id: E-state-machine-subgraph-name-default
title: "State machine sub-graph keeps engine default 'New State Machine' name — machineName renames the node but not its bound UEdGraph (states/conduits get renamed, the machine doesn't)"
status: OPEN
severity: Low
category: ergonomic
tags: [animation, anim-graph, state-machine, create_state_machine, add_state_machine, asset-dump, anim-graph-json, naming]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# `machineName` names the state-machine node but not its sub-graph page

Both `animation.create_state_machine` and the fine-grained
`animation.authoring.add_state_machine` name the `UAnimGraphNode_StateMachine`
**node** after the requested name (`machineName` / `stateMachineName`), but leave
the node's bound `UAnimationStateMachineGraph` (the sub-graph the user
double-clicks into — the editor tab, and the structured dump's `pages[]` entry)
at the engine default **"New State Machine"**.

So after asking for a state machine named `"Locomotion"`, the structured readback
disagrees with itself:
- `state_machines[].name` correctly reports `"Locomotion"` (the node).
- `pages[]` reports the sub-graph as `"New State Machine"` (the bound graph).

A consumer using `pages[]` to locate/navigate the state machine's sub-graph by
name gets a name that does not match what it asked for, and a human opening the
asset in-editor sees a tab labelled "New State Machine" rather than "Locomotion".
This is also an asset the editor itself would not produce: renaming a state
machine in the AnimGraph editor keeps its sub-graph tab in sync.

The inconsistency is internal to the construction helper. In
`AnimGraphConstructionUtils.cpp`, `CreateState` (`StateNode->BoundGraph`) and
`CreateConduit` (`ConduitNode->BoundGraph`) **both** call
`FBlueprintEditorUtils::RenameGraph(BoundGraph, *Name)` right after creation, so
states and conduits get their bound graphs renamed to match. `CreateStateMachine`
does **not**: it passes `Name` only as the *desired base name* to
`FBlueprintEditorUtils::CreateNewGraph(AnimBP, Name, UAnimationStateMachineGraph::StaticClass(), ...)`
and never follows up with a `RenameGraph` on the inner graph, so the inner graph
keeps the schema/engine default. States and conduits track the requested name;
the state machine alone does not.

**Workaround:** none needed for the node identity (`state_machines[].name` and
`agir.txt` both use the correct node name `"Locomotion"`). Treat the `pages[]`
entry whose `kind == "StateMachineGraph"` as the state machine regardless of its
`name`, and ignore the "New State Machine" label.

**Fix:** in `AnimGraphConstructionUtils::CreateStateMachine`, after binding
`SMNode->EditorStateMachineGraph = InnerGraph;`, call
`FBlueprintEditorUtils::RenameGraph(InnerGraph, *Name.ToString())` (mirroring the
existing `CreateState` / `CreateConduit` calls a few lines below) so the
sub-graph name tracks `machineName`. Then `pages[].name` and the in-editor tab
both read `"Locomotion"`.

## Verbatim repro

Skeleton precondition: `/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton` (exists).

1. `animation.create_animation_bp`
   `{name:"ABP_DinoDragon_Locomotion_OracleReplay", savePath:"/Game/AnimDev", skeletonPath:"/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton"}`
   → ok (`assetClass":"AnimBlueprint"`).
2. `animation.create_state_machine`
   `{blueprintPath:"/Game/AnimDev/ABP_DinoDragon_Locomotion_OracleReplay", machineName:"Locomotion", states:[{name:"Idle",isEntry:true},{name:"Move"}], transitions:[{sourceState:"Idle",targetState:"Move"}]}`
   → `{"blueprintPath":"...","machineName":"Locomotion","statesCreated":2,"transitionsCreated":1}` (success — the method itself works).
3. `asset.save {force:true}`, then `asset.dump`.

`anim_graph.json` — the node is `"Locomotion"` but its sub-graph page is `"New State Machine"`:

```json
"pages": [
    { "kind": "AnimGraph",          "name": "AnimGraph",         "parent": "" },
    { "kind": "StateMachineGraph",  "name": "New State Machine",  "parent": "" }
],
"state_machines": [
    { "name": "Locomotion", "page": "AnimGraph",
      "states": [ {"name":"Idle"}, {"name":"Move"} ],
      "transitions": [ {"from":"Idle","to":"Move", ...} ] }
]
```

Confirmed identical via the fine-grained path:
`animation.authoring.add_state_machine {blueprintPath:".../ABP_DinoDragon_Authoring_OracleReplay", stateMachineName:"Locomotion", x:300, y:200}` →
`{"nodeName":"Locomotion","success":true,"message":"State machine 'Locomotion' created with entry node"}`, yet its dumped `pages[]` again carries `{"kind":"StateMachineGraph","name":"New State Machine"}`. So the page-name default is shared in `AnimGraphConstructionUtils::CreateStateMachine`, not specific to `create_state_machine`.

## History
- `#1-initial-repro` `OPEN` reporter — REALISM-mode DinoDragon locomotion task. `animation.create_state_machine` (machineName `Locomotion`, states Idle/Move, transition Idle->Move) succeeds — `statesCreated:2, transitionsCreated:1` (the `B-create-state-machine-exec-failed` COMMAND_FAILED defect is no longer reproducible; the method is live). But the dumped `anim_graph.json` reports the node as `state_machines[].name == "Locomotion"` while the sub-graph `pages[]` entry (`kind == "StateMachineGraph"`) is named the engine default `"New State Machine"`. Confirmed the same default appears via the fine-grained `animation.authoring.add_state_machine`. Root cause located: in `AnimGraphConstructionUtils::CreateStateMachine` the inner `UAnimationStateMachineGraph` is created via `FBlueprintEditorUtils::CreateNewGraph(AnimBP, Name, ...)` and is never renamed, whereas the sibling `CreateState` / `CreateConduit` helpers in the same file both call `FBlueprintEditorUtils::RenameGraph(BoundGraph, *Name)` — so states and conduits track the requested name but the state machine sub-graph does not. Harm: the in-editor sub-graph tab and the structured `pages[].name` both read "New State Machine" instead of "Locomotion", contradicting the requested `machineName` and the asset the editor itself would produce on rename. Node identity (`state_machines[].name`, `agir.txt`) is correct, so functionally complete; this is a naming/readback inconsistency. Deduped: no existing ticket covers the SM sub-graph name (distinct from `B-create-state-machine-exec-failed` [the now-fixed dead-method bug], `E-anim-graph-json-omits-transition-logic-blend` [transition logic/blend dump fields, now surfacing], `F-anim-state-machine-internals` [DONE; rule-body/pose depth], `B-bpir-composite-inline-name-lost` [BPIR composite names]). Fix: add `FBlueprintEditorUtils::RenameGraph(InnerGraph, *Name.ToString())` after `SMNode->EditorStateMachineGraph = InnerGraph;` in `CreateStateMachine`, mirroring the existing state/conduit rename calls.
