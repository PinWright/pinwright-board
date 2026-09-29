---
id: F-animation-curve-editing-verbs
title: "animation.authoring cannot read curve keys, batch-write them, or remove / rename keys and curves"
status: IN-REVIEW
severity: Medium
category: feature
tags: [animation, curve, batch, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# animation.authoring cannot read curve keys, batch-write them, or remove / rename keys and curves

`animation.authoring` exposes `set_curve_key` (one key per call) and `list_curves` (names and key
counts only). An agent editing an AnimSequence curve cannot read the keys it is about to change,
cannot write a curve in one call, and cannot delete a key, delete a curve or rename a curve at all
without `python.execute`. The gap analysis of 2026-09-28 found other Unreal MCP servers expose
curve read and edit; PinWright had the write half of one key only.

`IAnimationDataController` already carries everything needed with identical signatures on UE 5.3
through 5.8: `SetCurveKeys`, `RemoveCurveKey`, `RemoveCurve`, `RenameCurve`, `AddCurve`,
`OpenBracket` / `CloseBracket` (`FScopedBracket`), and `FAnimationCurveIdentifier(FName,
ERawCurveTrackTypes)`. Two engine behaviours shape the verbs: `RenameCurve` does not check whether
the new name is taken (renaming onto an existing curve leaves two curves with one name), and
`RemoveCurveKey` matches the time with zero tolerance on 5.3 (`FindKey(Time, 0.f)`) but
`KINDA_SMALL_NUMBER` on later engines.

**Fix:** five verbs in `Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp`:
`get_curve_keys`, `set_curve_keys` (required `mode`: replace or merge), `remove_curve_key`,
`remove_curve`, `rename_curve`. Skeleton curve metadata is out of scope.

## History
- `#1-gap-analysis-filed` `OPEN` reporter — Filed from the 2026-09-28 gap analysis (plan task 10): no curve key read, no batch key write, no key or curve removal, no curve rename in `animation.authoring`. Supersedes the batch-write half of `F-animation-set-curve-key-no-batch`.
- `#2-five-curve-verbs` `IN-REVIEW` developer — Added `animation.authoring.get_curve_keys`, `set_curve_keys`, `remove_curve_key`, `remove_curve`, `rename_curve` in `Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp` (helpers in namespace `PwAnimCurveEdit`). Keys use one wire shape `{frame|time, value, interpMode, tangentMode, arriveTangent, leaveTangent}` for read and write; `keys` on `set_curve_keys` and `remove_curve_key` adopt the nested-key gate via a new `RPC_PARAM_REQ_NESTED` in `Handlers/ParamSpec.h` (adopters listed in `Tests/Infra/TestNestedParamKeyGate.cpp`). Multi-step edits run in one controller bracket, no `FScopedTransaction`. All input is validated before any write; removals are all-or-nothing. `rename_curve` refuses a taken `newName` with `ALREADY_EXISTS`; `remove_curve_key` passes the stored key time so 5.3's zero-tolerance match cannot miss. Responses report counts and keys read back from the data model. New code `CURVE_NOT_FOUND` in `Handlers/ErrorCodes.h`. No version guard needed (signatures identical 5.3-5.8). Tests `PinWright.animation.authoring.curve_editing.*` in `Tests/Gameplay/TestAnimationCurveEditing.cpp`; wiki `docs/wiki-src/animation.authoring.md`. Not compiled or run in this pass.
- `#3-cubic-after-linear-break` `IN-REVIEW` developer — Live check found a cubic key sent with no tangentMode reading back `break`. Root cause is the engine, not the parser or read mapping: the AnimationData plugin's sequencer data model (enabled by default, UE 5.3-5.8) converts keys through `AnimSequencerHelpers::ConvertRichCurveKeysToFloatChannel`, which forces `RCTM_Break` and a linear arrive tangent on any cubic key directly after a linear key, and the legacy curve is regenerated from that channel. The rewrite is what makes the linear segment evaluate correctly, so it is not undone. `set_curve_keys` now returns `engineAdjustedTangentModes: [{time, sent, stored}]` naming every sent key whose stored mode differs; the `keys` param description and wiki document the rewrite. Test `PinWright.animation.authoring.curve_editing.OmittedTangentModeRoundTrips` asserts omitted modes read back `auto` and a cubic-after-linear key is either `auto` or `break` and reported. Both touched .cpp files pass `-SingleFile` on 5.8; not run.
