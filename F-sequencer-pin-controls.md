---
id: F-sequencer-pin-controls
title: "No way to pin a Control Rig control (hand or foot contact) in place over a frame range"
status: IN-REVIEW
severity: Medium
category: feature
tags: [sequencer, control-rig, contact-lock, pose-editing, deferred, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Pin Control Rig controls over a frame range (contact lock)

**Deferred:** blocked on `F-sequencer-control-keys-batch`, whose transaction + readback + world-space
keying path this verb reuses.

No PinWright verb pins a control (a planted foot, a hand on a rail) over a range; the Control Rig
Sequencer verbs key and bake but do not hold a contact.

Competitors ("Pose editing aids" row): ue-mcp `contact_lock` op inside `apply_control_rig_edits`
(UE 5.8 only, `AnimationHandlers_ControlRigSequencer.cpp`): component-space target, fixed or sampled per
frame from a bone/socket of the source clip, smoothstep blend-in/out (`:1209-1251`); three solve paths —
direct pin with readback and undo on tolerance failure; rigid-offset back-solve with no readback
(`verification: bake_and_analyze_required`); FABRIK chain solve on FK rigs converted to rotation keys
with pre-commit residual prediction (`:1264-1321`, `:3359-3401`). Epic's toolset has `snap_control_rig`
and space baking. Monolith's `enable_foot_ground_lock` only edits the retargeter Speed Planting op and
is disabled on 5.8.

## Engine API

`ControlRigSequencerEditorLibrary.h` (UE 5.8): `GetControlRigWorldTransforms` `:570`,
`SetControlRigWorldTransforms` `:597` (need an open, focused Sequencer), `SnapControlRig` `:443`,
`AddConstraint` / `SetConstraintActiveKey` / `BakeConstraint` `:320-364`. No engine IK solve for FK
chains (ue-mcp calls `AnimationCore::SolveFabrik` itself).

## Proposed scope (v1)

`sequencer.pin_controls {sequence, binding, controls[] (required), startFrame, endFrame (required), target: "anchorFrame"|{transform} (required), anchorFrame, blendInFrames, blendOutFrames, positionToleranceCm, rotationToleranceDeg}`:
- Read world transforms, compute blended targets, write with `SetControlRigWorldTransforms` in one transaction.
- Read back; report `maxErrorCm` / `maxErrorDeg` per control; undo and return `CONTACT_TOLERANCE_EXCEEDED` when outside tolerance.
- Document that v1 moves only the pinned control: correct for IK/effector controls, not for FK chains. FK chain solve is v2.

Pose propagation (captured viewport pose applied across frames) is deliberately out of scope;
`sequencer.get_control_values` + `set_control_keys` cover "apply this delta at these frames".

## Acceptance

- A foot IK control holds within 0.1 cm over 30 frames.
- Blend weights verified at the edge frames.
- A tolerance failure undoes all changes.

**Effort:** M (L with the FK chain solve).

## History
- `#1-no-contact-pin` `OPEN` reporter — Filed from the 2026-09-30 animation gap analysis at plugin HEAD `2580e7f4`; deferred behind `F-sequencer-control-keys-batch` (gate set in `blockedBy`). Severity Medium.
- `#2-pin-controls-v1` `IN-REVIEW` developer — `F-sequencer-control-keys-batch` is implemented, so `blockedBy` is dropped. Added `sequencer.pin_controls` (`controls[]`, `startFrame`, `endFrame`, `target: "anchorFrame" | {control: worldTransform}` required; `anchorFrame` required iff anchor target; `blendInFrames`/`blendOutFrames` keyed outside the hold with smoothstep weight 0 at the outer edge, 1 across the hold; `positionToleranceCm`/`rotationToleranceDeg` default 0.01). Reads original world motion and anchors first, then writes through the same headless world-space path and transaction/readback/rollback as `set_control_keys`; per control `maxErrorCm`/`maxErrorDeg` against the blended targets; any key over tolerance undoes everything and fails `CONTACT_TOLERANCE_EXCEEDED` (`rolledBack`). v1 moves only the pinned controls (documented: right for IK/effector controls, not FK chains; FK chain solve stays v2). Files: `Handlers/Sequencer/ControlRigSequencerHandler.cpp`, `Handlers/ErrorCodes.h` (+`CONTACT_TOLERANCE_EXCEEDED`), `docs/wiki-src/sequencer.md`, `Tests/Sequencer/TestSequencerControlRigKeyBatch.cpp`. Tests: `PinWright.Sequencer.ControlRigKeys.PinHoldsAndBlends` (moving parent; Tip holds within 0.1 cm over 30 frames; blend weights at frames 15/17/52/54), `.PinToleranceFailureUndoesAll` (injected 5 cm readback error; every channel restored exactly, package dirty flag restored). Not run yet.
