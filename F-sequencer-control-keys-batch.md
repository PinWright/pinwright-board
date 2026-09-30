---
id: F-sequencer-control-keys-batch
title: "Control Rig keying is one frame per call and readback one control and one frame per call; no world-space keying and no documented clip-edit workflow"
status: OPEN
severity: Medium
category: feature
tags: [sequencer, control-rig, keying, batch, animation, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Batch Control Rig key/read, plus the non-destructive clip-edit workflow

The pieces for editing an AnimSequence through Control Rig without touching the source already exist:
`sequencer.create` → bind a skeletal actor → `sequencer.add_animation_track` →
`sequencer.bake_to_controlrig` → `sequencer.key_controls` → `sequencer.export_anim_sequence` (never
links back into the sequence). The friction:
- `sequencer.key_controls` keys one frame per call (`Source/PinWright/Private/Handlers/Sequencer/ControlRigSequencerHandler.cpp:680`).
- `sequencer.get_control_value` reads one control at one frame per call.
- No world/component-space keying.
- No wiki page describes the chain, and a SkeletalMesh asset cannot be added as a spawnable (`E-sequencer-add-spawnable-class-only-not-mesh-asset`, OPEN).

Competitor ("Non-destructive Control Rig sessions" row): ue-mcp `begin/read/apply/bake_control_rig_edit`
(UE 5.8 only). `apply_control_rig_edits` ops `set`, `set_keys`, `offset` (blend ramp), `contact_lock`,
`set_bool/float/int`, `propagate_pose`; prepares all writes, commits in one `FScopedTransaction`, reads
every key back and calls `UndoTransaction` on mismatch
(`AnimationHandlers_ControlRigSequencer.cpp:3547-4001`). A server-side session is not needed to get the
same safety; batch verbs are.

## Engine API

`C:/UE_5.8/Engine/Plugins/Animation/ControlRig/Source/ControlRigEditor/Public/ControlRigSequencerEditorLibrary.h`:
`BatchGetControlTransforms` `:529`, `BatchSetControlTransforms` `:544`,
`Get/SetLocalControlRigTransforms` `:1103/:1130`, `Get/SetControlRigWorldTransforms` `:570/:597`
(world variants need an open, focused Sequencer: `LocalSetControlRigWorldTransforms` calls
`GetSequencerFromAsset`; reuse the focus path `sequencer.bake_control_space` already has).

## Proposed scope

- `sequencer.set_control_keys {sequence, binding, keys: [{control, frames[], values[][]}] (required), space: local|world (required)}` — all keys in one transaction, read back, undo everything and return `CONTROL_KEY_READBACK_MISMATCH` on any mismatch; report per-control keys written and max readback error.
- `sequencer.get_control_values {sequence, binding, controls[], frames[], space (required)}`.
- A workflow wiki page: "edit a clip through Control Rig, export a new clip", covering the chain above.

## Acceptance

- 3 controls × 20 frames keyed in one call.
- An injected readback mismatch undoes every key.
- `world` space round-trips within tolerance.
- The workflow page's chain runs end to end and leaves the source AnimSequence byte-unchanged.

**Effort:** M.

**Related:** `F-sequencer-controlrig-track` (IN-REVIEW, delivered single-frame keying), `E-sequencer-add-spawnable-class-only-not-mesh-asset` (prerequisite for the workflow), `F-sequencer-pin-controls` (builds on this).

## History
- `#1-single-frame-keying` `OPEN` reporter — Filed from the 2026-09-30 animation gap analysis at plugin HEAD `2580e7f4`. Severity Medium: doable today only with one call per control per frame.
