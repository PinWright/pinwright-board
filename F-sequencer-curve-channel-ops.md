---
id: F-sequencer-curve-channel-ops
title: "sequencer.add_keyframe writes cubic-only — no caller-selectable interp/tangent (can't author eased/held/stepped keys)"
status: IN-REVIEW
severity: Medium
category: feature
tags: [sequencer, curves, keyframes, interp, tangent, parity-ue58]
---

# `sequencer.add_keyframe` writes cubic-only — no caller-selectable interp/tangent

`sequencer.add_keyframe` writes every float key with hardcoded cubic interpolation:
`SequencerHandler.cpp:162` calls `FMovieSceneFloatChannel::AddCubicKey(FrameNumber, Value)`
with no interp/tangent control, and its param spec (`:57-63`) exposes only
`sequencePath`/`bindingGuid`/`propertyName`/`time`/`value`. The ControlRig keyer hardcodes
cubic the same way (`ControlRigSequencerHandler.cpp:622`, `KeyValue.InterpMode = RCIM_Cubic`).
So RPC-authored keys always land cubic — agents cannot author **constant** (holds/stepped) or
**linear** keys, nor pick a cubic **tangent mode** for eases, which makes generated cinematics
look mechanical.

The channel readback also cannot distinguish interp modes: the opt-in `includeKeys` payload
(`MovieSceneJsonUtils::BuildChannelKeysJson`, added by the IN-REVIEW
`F-sequencer-list-channels-keyframe-readback`) emits only `{frame, value}` per key, never the
interp mode — so even a correctly-authored linear/constant key is indistinguishable from cubic
through any sequencer readback.

## Scope (reworded — was a 3-feature grab-bag)

Originally filed bundling three orthogonal features (interp/tangent + a batch `edit_keys` +
level-sequence **marked frames**). Only the interp/tangent gap serves the reporter's stated
complaint ("cinematics look mechanical"): marked frames are editor bookmarks on `UMovieScene`
that do not affect playback or interpolation, and batch key-edit is unrelated convenience —
neither was driven by a real task. **Narrowed to the interp/tangent core plus the per-key interp
readback needed to verify it.** Marked frames and batch key-edit are dropped; re-file them as
their own tickets if a real task hits them. (The legacy frame-based `sequence.add_keyframe`
dual-shape quirk remains out of scope, as originally noted.)

Precedent to mirror: `niagara.set_curve_keys` already ships per-key `interp`
(constant|linear|cubic) + tangent authoring (`NiagaraCurveHandler.cpp:169-178` `ParseInterpMode`);
this uses the same shape.

**Fix:** Add optional `interp` (constant|linear|cubic, default cubic — byte-identical back-compat)
and `tangentMode` (auto|user|break|none) params to `sequencer.add_keyframe`, writing via
`AddConstantKey`/`AddLinearKey`/`AddCubicKey(…, TangentMode)` and echoing the applied `interp`.
Extend `BuildChannelKeysJson` to emit per-key `interp` for float/double channels under the
existing `includeKeys` gate so the round-trip is verifiable (default-off payload + asset dump
unchanged, no aspect bump).

Acceptance: author constant/linear/cubic keys on one float channel via `sequencer.add_keyframe`;
the stored `FMovieSceneFloatValue.InterpMode` matches, and `sequencer.list_sections includeKeys=true`
reports the per-key `interp`.

## History
- `#1-default-interp-only` `OPEN` reporter — Keys land with default interpolation; no tangent/interp/batch/marked-frame ops (greps verified). Epic 5.8 keyframing suite has channel-level control; file interp/batch/marked-frame RPCs.
- `#2-reword-interp-core` `IN-REVIEW` developer — Reworded the 3-feature grab-bag down to the interp/tangent core (dropped the marked-frames + batch `edit_keys` riders — neither served the reporter's "mechanical cinematics" complaint nor was driven by a task; re-file separately if hit; original grep `SequencerHandler.cpp:136` cite corrected to the real write at `:162`). Implemented caller-selectable interpolation on `sequencer.add_keyframe`: new optional `interp` (constant|linear|cubic, default cubic = byte-identical to the old `AddCubicKey`) and `tangentMode` (auto|user|break|none) params, written via `AddConstantKey`/`AddLinearKey`/`AddCubicKey(…, TangentMode)` and echoed as `interp` (`Handlers/Sequencer/SequencerHandler.cpp`; parse helpers in the file-local `SequencerSectionHelpers`). Made it verifiable through readback: `MovieSceneJsonUtils::BuildChannelKeysJson` (`Utils/MovieSceneJsonUtils.h`) now emits per-key `interp` for float/double channels under the existing opt-in `includeKeys` gate (default-off list_sections payload + asset dump unchanged → no aspect bump). Modeled on the shipped `niagara.set_curve_keys` per-key interp shape. Regression test `PinWright.Sequencer.AddKeyframe.InterpModeApplied` (`Private/Tests/Sequencer/TestSequencerAddKeyframeInterp.cpp`) authors constant + linear + cubic keys via the production handler and asserts the stored `InterpMode` per key plus the `includeKeys` readback `interp` string — both halves fail pre-fix (all keys landed cubic; no interp field in readback).
