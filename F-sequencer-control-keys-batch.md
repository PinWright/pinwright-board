---
id: F-sequencer-control-keys-batch
title: "Control Rig keying is one frame per call and readback one control and one frame per call; no world-space keying and no documented clip-edit workflow"
status: IN-REVIEW
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
- `#2-batch-world-keys` `IN-REVIEW` developer — Added `sequencer.set_control_keys` (`keys: [{control, frames, values}]`, `space: local|world` required; one FScopedTransaction; touched channels snapshotted; every key read back — local to float precision, world re-posed through the rig against `positionToleranceCm`/`rotationToleranceDeg` (default 0.01) and scale 0.001; any mismatch restores the snapshot, cancels the transaction, verifies the restore and fails `CONTROL_KEY_READBACK_MISMATCH` with `mismatches` + `rolledBack`) and `sequencer.get_control_values` (`controls[] x frames[]`, local channel values or world `[TX,TY,TZ,Roll,Pitch,Yaw,SX,SY,SZ]`). World space is computed headless instead of through the engine's open-Sequencer `Get/SetControlRigWorldTransforms`: pose the rig from the section's channels at the frame, run the forward solve when the control hangs off a bone (FK), world = control global x bound component world; refused `CONTROL_WORLD_SPACE_UNAVAILABLE` for additive rigs, bindings (or parents) with a transform/attach track, or no bound component. Workflow page `docs/wiki-src/sequencer.controlrig-clip-edit.md`; method sections in `docs/wiki-src/sequencer.md`. Files: `Source/PinWright/Private/Handlers/Sequencer/ControlRigSequencerHandler.cpp` (new verbs; file adopted the ErrorCodes registry), `Handlers/Sequencer/ControlRigKeyTestHooks.h` (new), `Handlers/ErrorCodes.h` (+`CONTROL_KEY_READBACK_MISMATCH`, `CONTROL_WORLD_SPACE_UNAVAILABLE`), `Tests/Sequencer/TestSequencerControlRigKeyBatch.cpp` (new). Tests: `PinWright.Sequencer.ControlRigKeys.LocalThreeControlsTwentyFrames`, `.ReadbackMismatchUndoesEveryKey`, `.WorldSpaceRoundTrip`, `.ClipEditWorkflowLeavesSourceUnchanged` (mannequin content; source `.uasset` MD5 unchanged). Not run yet (manager owns the editor slot); compile-checked with UBT -SingleFile.
- `#3-fixround-binding` `IN-REVIEW` developer — Full-suite run (w23): `LocalThreeControlsTwentyFrames`, `ReadbackMismatchUndoesEveryKey` and `ClipEditWorkflowLeavesSourceUnchanged` passed (the last includes a world-space key on an FK `hand_r` control within 0.1 cm, so the FK forward-solve path is proven). `WorldSpaceRoundTrip` / `PinHoldsAndBlends` / `PinToleranceFailureUndoesAll` failed `CONTROL_WORLD_SPACE_UNAVAILABLE`: the fixture bound its rig to a static-mesh component, and `FControlRigObjectBinding::GetBindableObject` only binds skeletal-mesh / Control Rig components, so the rig had no bound object. That was a fixture bug, not a verb bug; the fixture now binds a mesh-less `ASkeletalMeshActor`. `set_control_keys:keys` added to `PinWright.infra.dispatcher.NestedParamKeyGate.AdoptionSetIsRatcheted` and its description now says the entry schema is closed (UNKNOWN_NESTED_PARAMS).
