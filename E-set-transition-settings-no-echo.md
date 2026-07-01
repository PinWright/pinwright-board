---
id: E-set-transition-settings-no-echo
title: "animation.authoring.set_transition_settings success response echoes none of logicType/blendMode/disabled it just applied — forces an asset.dump readback round-trip to confirm the write"
status: OPEN
severity: Low
category: ergonomic
tags: [animation, animation-authoring, set-transition-settings, readback, round-trip, response-shape, echo, consistency]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `set_transition_settings` doesn't echo what it set, so every "did it apply?" check costs a separate dump round-trip

`animation.authoring.set_transition_settings` parses and applies up to
three transition fields — `logicType` (`StandardBlend`/`Inertialization`/
`Custom`), `blendMode` (`EAlphaBlendOption`), and `disabled` — but its
success response carries **none of them**. It returns only
`{"fromState":"...","toState":"...","success":true}`
(`AnimationAuthoringHandler_AnimBlueprint.cpp:1131-1135`), discarding the
exact values the handler had in hand one line earlier (`LogicValue`
applied at :1091, `BlendValue` at :1102, `bDisabled` at :1119). A caller
therefore cannot confirm the write succeeded *with the intended values*
from the response — it has to read the transition back out of the asset.

This is the same "the mutator could carry the answer" response-shape gap
as `E-pcg-add-node-echo-pin-labels` (create handlers omit pin labels) and
`E-geometry-deformer-echo-mesh-counts` (deformers omit vertex/triangle
counts): the value the caller needs to verify the op is trivially
available at response-build time and instead forces a follow-up read.

## Why it's process friction (distinct from the judge's ticket)

The judge filed `E-anim-graph-json-omits-transition-logic-blend` for the
**readback surface**: that `asset.dump`'s `anim_graph.json` omits
`logicType`/`blendMode` from its `transitions[]` schema, so the readback
*also* fails to show the values cleanly. That ticket's fix is to extend
`AnimGraphDumpBuilder`'s dump schema.

This ticket is the **other half / cheaper fix**: have the authoring RPC
**echo what it set in its own response**, which eliminates the readback
round-trip entirely. The two are complementary fix surfaces — the
authoring handler's response JSON vs. the dump builder's sidecar schema —
and either one alone closes the loop for the common single-transition
case:
- Echo in the response (this ticket): the agent confirms the write from
  the same call. **No `asset.dump`, no `agir.txt`, no enum decode.**
- Fix the dump (judge's ticket): the structured readback works, but the
  agent still pays a separate `asset.dump` call and parses a sidecar.

The response echo is the lower-cost fix (a few `SetStringField`/
`SetBoolField` calls on a `Result` object the handler already builds) and
the one that removes the round-trip rather than just making it succeed.

## What it should do

`set_transition_settings`'s success response should echo the fields it
actually applied — `logicType` and `blendMode` as their **string input
vocabulary** (`"Inertialization"`, `"Cubic"` — not raw enum indices) and
`disabled` as a bool — only for the params that were present in the call
(it's an "update advanced properties" RPC where each param is optional, so
echo only what was set, matching the optional-param semantics at
:1003-1006). The handler already holds `LogicValue`/`BlendValue`/
`bDisabled`; mapping them back to the canonical strings (the same tables
used to parse them in) is cheap and keeps the echo consistent with the
input vocabulary.

**Workaround:** read the transition back via `asset.dump` and decode
`agir.txt`'s `logic_type`/`blend_mode` raw enum indices by hand (the
structured `anim_graph.json` omits them — see
`E-anim-graph-json-omits-transition-logic-blend`).

## Friction evidence (this task — focus `set_transition_settings`, 15 calls, outcome clean/ergo)

Friction note verbatim: *"set_transition_settings has no dedicated
get/dump method and its own success response doesn't echo values, so I had
to verify via asset.dump's agir.txt (logic_type=1/blend_mode=1) and decode
the numeric enum indices against the engine's ETransitionLogicType and
EAlphaBlendOption headers; anim_graph.json showed disabled but omitted
logicType/blendMode entirely."*

The task built `ABP_DinoDragon_Locomotion` and the story's step 7
explicitly required reading back that `Walk->Run` ended up
Inertialization/Cubic and `Run->Idle` ended up disabled. Because the two
`set_transition_settings` calls returned only `{fromState,toState,success}`,
the confirm step forced a trailing `asset.dump` (writing `anim_graph.json`
+ `agir.txt`) and then a manual enum-index decode — the round-trip the
seed method's own response could have made unnecessary. The
`set_transition_settings` seed landed clean in the ledger; this is pure
PROCESS overhead, not an outcome bug.

Distinct from `E-anim-graph-json-omits-transition-logic-blend` (judge's —
the dump readback surface) and from `F-anim-state-machine-internals` (DONE
— the feature that *added* `set_transition_settings`). Recurs once per
transition whose advanced settings a caller needs to verify.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `animation.authoring.set_transition_settings` `ABP_DinoDragon_Locomotion` locomotion-state-machine task (15 calls, outcome clean/ergo; judge filed `E-anim-graph-json-omits-transition-logic-blend` for the dump-schema half). PROCESS finding: `set_transition_settings` applies `logicType`/`blendMode`/`disabled` (parsed at AnimationAuthoringHandler_AnimBlueprint.cpp:1071-1124) but its success response (:1131-1135) echoes only `{fromState,toState,success}` — none of the three values, all of which the handler holds at response-build time. The story's step-7 confirm therefore forced a trailing `asset.dump` + manual decode of `agir.txt`'s raw `logic_type=1`/`blend_mode=1` indices. Distinct fix surface from the judge's ticket: echo the applied values (as their string input vocabulary) in the RPC response to remove the readback round-trip entirely, rather than only making the dump readback succeed. Same "mutator should carry the answer" shape as `E-pcg-add-node-echo-pin-labels` and `E-geometry-deformer-echo-mesh-counts`. Severity Low (recoverable via dump; never blocks). Deduped: no existing ticket covers the `set_transition_settings` response-echo angle — the only sibling, `E-anim-graph-json-omits-transition-logic-blend`, targets `anim_graph.json`'s dump schema, not the authoring RPC's response.
