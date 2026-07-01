---
id: E-anim-graph-json-omits-transition-logic-blend
title: "anim_graph.json transition schema omits logicType/blendMode — set_transition_settings outputs are unreadable in the structured dump"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [animation, asset-dump, anim-graph, state-machine, transition, readback, set-transition-settings]
---

# `anim_graph.json` reports `disabled` but silently omits `logicType`/`blendMode`

`animation.authoring.set_transition_settings` writes three transition
fields: `logicType` (`StandardBlend`/`Inertialization`/`Custom`),
`blendMode` (`EAlphaBlendOption`), and `disabled`. Its own success
response echoes **none** of them — it returns only
`{"fromState":"...","toState":"...","success":true}`.

So to confirm what was applied, a caller has to read it back. The
canonical structured readback for an anim BP is `asset.dump`'s
`anim_graph.json` sidecar, whose `state_machines[].transitions[]` array
already carries per-transition fields. But that array emits only
`bidirectional`, `disabled`, `from`, `priority`, `rule_graph`, `to` — it
**omits `logicType`/`blendMode` (the AGIR `logic_type`/`blend_mode`)
entirely**. The very two advanced fields that `set_transition_settings`
exists to author are invisible in the structured JSON, while the third
field it writes (`disabled`) is present.

The net ergonomic harm: after setting `logicType=Inertialization,
blendMode=Cubic` on a transition, the structured readback shows that
transition with no logic/blend fields at all — indistinguishable from a
default/unset transition. A consumer comparing "what I set" against the
JSON would reasonably conclude the call silently no-op'd, when in fact it
applied correctly. The values are only recoverable from the sibling
`agir.txt` text IR, and there only as **raw enum indices** the caller
must decode against the engine's `ETransitionLogicType` /
`EAlphaBlendOption` headers (`logic_type=1` = Inertialization,
`blend_mode=1` = Cubic).

This is a schema-drift artifact: `anim_graph.json`'s transition schema
was frozen with `priority/rule_graph/bidirectional/disabled` (see
`F-bpir-animgraph-not-walked` history `#3`) *before*
`set_transition_settings` (`F-anim-state-machine-internals` `#3`) added
the ability to author `logicType`/`blendMode`. The readback aspect was
never extended to surface the two new writable fields.

**Workaround:** read `agir.txt` and decode `logic_type` / `blend_mode`
numeric indices by hand against the engine enums.

**Fix:** add `logicType`/`blendMode` (string names, matching
`set_transition_settings`'s input vocabulary — not raw indices) to each
`anim_graph.json` `transitions[]` entry in `AnimGraphDumpBuilder`, so the
structured dump round-trips every field the authoring RPC writes.
`blendCurvePath` should follow for the `Custom` logic case.

## Verbatim repro

Built `ABP_DinoDragon_Locomotion_Replay` (state machine `Locomotion`;
states `Idle`/`Walk`/`Run`; transitions `Idle->Walk`, `Walk->Run`,
`Run->Idle`), then:

1. `animation.authoring.set_transition_settings`
   `{blueprintPath:"/Game/ExampleContent/IKRig/Anim/ABP_DinoDragon_Locomotion_Replay", stateMachineName:"Locomotion", fromState:"Walk", toState:"Run", logicType:"Inertialization", blendMode:"Cubic"}`
   → `{"fromState":"Walk","toState":"Run","success":true}` (no echo of the values set).
2. `animation.authoring.set_transition_settings`
   `{... fromState:"Run", toState:"Idle", disabled:true}`
   → `{"fromState":"Run","toState":"Idle","success":true}`.
3. `asset.dump {assetPath:"/Game/ExampleContent/IKRig/Anim/ABP_DinoDragon_Locomotion_Replay"}`.

`anim_graph.json` — the `Walk->Run` transition carries no logic/blend fields:

```json
{ "bidirectional": false, "disabled": false, "from": "Walk",
  "priority": 1, "rule_graph": "Transition", "to": "Run" }
```

(`Run->Idle` correctly shows `"disabled": true` — proving the schema can
carry `set_transition_settings` outputs; it just doesn't carry the two
blend fields.)

`agir.txt` — same two transitions, where the values actually surface (as raw indices):

```
transition Walk -> Run priority=1 rule=Transition crossfade_duration=0.2 blend_mode=1 logic_type=1 ...
transition Run -> Idle priority=1 rule=Transition disabled=true crossfade_duration=0.2 blend_mode=0 logic_type=0 ...
```

## History
- `#1-initial-repro` `OPEN` reporter — `set_transition_settings` writes `logicType`/`blendMode`/`disabled` but its success response echoes none of them, and the structured readback `asset.dump` → `anim_graph.json` `transitions[]` carries `bidirectional/disabled/from/priority/rule_graph/to` while omitting `logicType`/`blendMode` entirely. Verified on `ABP_DinoDragon_Locomotion_Replay`: after setting Walk->Run to Inertialization/Cubic, `anim_graph.json` shows that transition with no logic/blend fields (indistinguishable from default), while `agir.txt` exposes them only as raw enum indices (`blend_mode=1 logic_type=1`) requiring manual decode against `ETransitionLogicType`/`EAlphaBlendOption`. `disabled` round-trips correctly (Run->Idle shows `"disabled": true`), proving the schema can carry these outputs. Fix: add string-named `logicType`/`blendMode` (and `blendCurvePath` for Custom) to `anim_graph.json` transition entries in `AnimGraphDumpBuilder`.
- `#2-fix` `IN-REVIEW` developer — Extended the `anim_graph.json` transition schema to surface every field `set_transition_settings` writes. In `AnimGraphDumpBuilder.cpp` each `state_machines[].transitions[]` entry now emits `logicType` (string `StandardBlend`/`Inertialization`/`Custom`, the RPC's own input vocabulary — engine literal with the `TLT_` prefix stripped via a new `LogicTypeToString` helper) and `blendMode` (the `EAlphaBlendOption` enum-literal name via `StaticEnum<>()->GetNameStringByValue`, matching `ParseAlphaBlendOption`'s input vocabulary, new `BlendModeToString` helper), plus `blendCurvePath` (asset path of `CustomBlendCurve`) when `LogicType==Custom` and a curve is set. Added the `Animation/AnimStateMachineTypes.h` / `AlphaBlend.h` / `Curves/CurveFloat.h` includes. Bumped the `anim_graph.json` aspect version (added entry =2, was the default 1) in `AssetDumpCache.cpp` so stale dumpcache entries are invalidated and regenerated with the new schema. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/AnimGraphDumpBuilder.cpp`, `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetDumpCache.cpp`. Test: extended `FAnimAuthoringStateMachineInternalsTest` (`EditorAutomationRpcGateway.anim.authoring.StateMachineInternals`, `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestAnimGraphHandlers.cpp`) — after `set_transition_settings` writes `logicType=Inertialization,blendMode=Cubic` on the Idle->Run transition, it now calls the production `AnimGraphDumpBuilder::BuildAnimGraphJson(AnimBP)` and asserts the dumped transition reports `logicType==Inertialization` and `blendMode==Cubic`; reverting the dump-builder change drops those fields and the new assertions fail. Sibling response-echo gap stays scoped to `E-set-transition-settings-no-echo`.
