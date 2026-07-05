---
id: F-sequencer-curve-channel-ops
title: "Sequencer channel-level key editing: interp/tangent modes, batch edit, marked frames"
status: OPEN
severity: Medium
category: feature
tags: [sequencer, curves, keyframes, parity-ue58]
---

# Sequencer channel-level key editing: interp/tangent modes, batch edit, marked frames

Keyframes are written with default interpolation only: `SequencerHandler.cpp:136` adds keys on `FMovieSceneFloatChannel` with no tangent/interp control (grep `SetInterpMode|ERichCurveInterpMode|Tangent` in `Source\PinWright\Private\Handlers\Sequencer\` = zero). There is also no marked-frames support (`MarkedFrame` = zero) and no batch key edit. Agents cannot author eases, holds, or stepped keys, which makes generated cinematics look mechanical.

UE 5.8 parity evidence: AnimationAssistantToolset keyframing suite (22 tools, `...\animation_toolset\toolsets\keyframing.py`) exposes channel-level keyframing with tangent/interp control and curve-editor operations; sequencer suite includes marked frames.

Related (readback side, do not duplicate): `F-sequencer-list-channels-keyframe-readback` (IN-REVIEW). Related hygiene, NOT in scope here: the documented dual `add_keyframe` shape quirk (`sequence.add_keyframe` frame-based vs `sequencer.add_keyframe` seconds-based float-only, see wiki sequencer.md).

Proposed scope:
- `sequencer.set_key_interp(sequence, binding, track, channel, keys[], interp, tangentMode)`.
- `sequencer.edit_keys` - batch move/scale/set-value on selected keys of a channel.
- `sequencer.add_marked_frame` / `list_marked_frames` / `remove_marked_frame`.

Acceptance: author cubic/linear/constant keys on one channel and verify via the channel readback RPC; marked frame round-trips through list.

## History
- `#1-default-interp-only` `OPEN` reporter — Keys land with default interpolation; no tangent/interp/batch/marked-frame ops (greps verified). Epic 5.8 keyframing suite has channel-level control; file interp/batch/marked-frame RPCs.
