---
id: F-pose-repeatability-checkpoint
title: "Time-driven pose captures cannot measure repeatability because no subject-state checkpoint can replay pose zero"
status: IN-REVIEW
severity: Medium
category: feature
tags: [render, capture, pose-set, repeatability, subject-time, checkpoint]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# Time-driven pose captures cannot measure repeatability

## What happens

Pose-set capture scans the request for any `SubjectTimeSeconds` value and refuses
the control re-shot when one is present because replaying the setter may advance
simulation state (`Source/PinWright/Private/Handlers/Render/PoseListCapture.cpp:390-408`).
Pixel comparison runs only for non-time-driven sets (`PoseListCapture.cpp:409-455`),
and the response consequently reports `poseRepeatability.measured:false` with the
reason (`PoseListCapture.cpp:528-560`).

## Why it matters

For animated, Niagara, or other time-driven subjects, callers cannot tell whether
two frames differ because rendering is unstable or because the subject advanced.
Severity is Medium: a normal capture can still be produced, but the repeatability
acceptance question has no reliable in-call answer.

## What should happen

Add a subject-state checkpoint/rewind/restore contract that can return to the exact
pose-zero state without advancing it, take the control capture from that checkpoint,
and restore the caller's original state on every exit. Report explicitly when a
subject type cannot provide such a checkpoint.

## Workaround

Reset the subject externally and compare independent capture calls. This cannot
guarantee the same state for non-idempotent time setters.

## Related

- `B-capture-render-resolution-unreported` — wave-6 capture ticket whose primitive
  review exposed the missing time-driven repeatability checkpoint.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed the subject-time gate at `PoseListCapture.cpp:390-408`, the skipped control comparison at `:409-455`, and the `measured:false` response shape at `:528-560`. No capture, build, test, editor, or MCP call was run. Severity Medium because capture remains possible, but repeatability for time-driven subjects is not measurable without an external, unreliable reset.
- `#2-subject-state-checkpoint-seam` `IN-REVIEW` developer — Added a subject state checkpoint seam: `FSubjectStateCheckpointer` / `FSubjectStateRestorer` (`PoseListCapture.h`, same types on `CaptureSubject.h`) plus `FPoseListCaptureRequest::SubjectStateCheckpointer` / `SubjectStateCheckpointUnavailableReason` and `FResolvedSubject::StateCheckpointer` / `StateCheckpointUnavailableReason`. `RunPoseListCapture` (`PoseListCapture.cpp`) now, for a set that drove subject time, checkpoints the subject immediately before pose 0's real shot and again at the end of the set, rewinds to the pose-0 checkpoint for the control frame (the time setter is never replayed), and puts the end-of-set checkpoint back on every exit after a rewind was attempted, including a failed rewind or failed control. A kind with no checkpointer gets `measured:false` with a `notMeasuredReason` that carries the provider's reason. New response block `poseSet.poseRepeatability.subjectCheckpoint {available, unavailableReason?, rewoundForControl, restoredAfterControl?, restoreWarning?}`, present only on time-driven sets. Providers: scrubbed animation and skeletal-mesh previews checkpoint the paused single-node position (`MakeScrubStateCheckpointer`, `CaptureSubjectProviders_Animation.cpp`, bound in `_Animation.cpp` / `_Mesh.cpp` and in `AnimationPreviewCaptureHandler.cpp`); Level Sequence subjects checkpoint the paused playhead (`CaptureSubjectProviders_Level.cpp`); Niagara binds none and says why (`CaptureSubjectProviders_Niagara.cpp`). Wired in `RenderHandler.cpp` (render.capture_asset_preview), `AnimationShotsHandler.cpp` (camera.animation_shots), `AnimationPreviewCaptureHandler.cpp` (render.capture_animation_preview). Parity rows for the two new request fields in `TestCaptureVerbParameterParity.cpp`. Docs: `docs/wiki-src/render.md` (poseRepeatability paragraph and key table), CHANGELOG. Tests (injected primitive, no RHI, no skip path): `PinWright.render.pose_list.subject_checkpoint.RewindsTimeDrivenControl`, `.UnavailableKindReportsReason`, `.RestoresEndStateOnEveryExit` (new file `Tests/Render/TestPoseListCaptureSubjectCheckpoint.cpp`). Not live-verified in a real Persona/Sequencer viewport; the scrub restore also recreates the render proxy (`MarkRenderStateDirty`) to avoid the coalesced-update stale pose.
- `#3-review-fixes` `IN-REVIEW` developer — Review a3b19f6111bd0190e. (1) `CaptureSubjectProviders_Level.cpp`: the playhead checkpoint restorer now always applies `EUpdatePositionMethod::Jump` (keeps `bForceUpdate`) instead of the caller's `updateMethod`, so under `camera.animation_shots updateMethod:'play'` the rewind and put-back no longer sweep the range with Play semantics and fire event tracks. Both the checkpointer and its restorer now refuse with `SEQUENCE_NOT_OPEN` when `GetCurrentLevelSequence()` is no longer the subject's sequence (was: unconditional `true`). (2) `TestPoseListCaptureSubjectCheckpoint.cpp`: `RestoresEndStateOnEveryExit` gains a failed-rewind block (`FailRestoreOfCheckpoint = 0`; the stub restorer now moves `State` before refusing) asserting `State == 6`, `!bSubjectRewoundForControl`, `!bPoseRepeatabilityControlShotTaken`, `bSubjectRestoredAfterControl`. Docs: dropped the stale "for non-time-driven sets" qualifier in `docs/wiki-src/render.md` and the matching `PoseListCapture.cpp` comment; CHANGELOG notes the Jump rule. The Jump fix has no automated failure-direction test (needs a live Sequencer event-track fixture); fastcheck OK on the three touched .cpp files.
- `#4-linux-verification` `IN-REVIEW` tester — run3/full on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d), all non-skipped: `PinWright.render.pose_list.subject_checkpoint.RewindsTimeDrivenControl`, `.UnavailableKindReportsReason` and `.RestoresEndStateOnEveryExit`, which includes the failed-rewind block. These prove the primitive's contract with an injected stub: checkpoint before pose 0, rewind for the control shot without replaying the time setter, restore the end-of-set state on every exit, and `measured:false` with the provider's reason when no checkpointer exists. Remaining: no test exercises a real provider. The scrub checkpointer for animation and skeletal-mesh previews, the Level Sequence playhead checkpointer and its always-Jump restore (developer #3 says the Jump fix has no failure-direction test) were not run in a real Persona or Sequencer viewport. So `poseRepeatability.measured:true` on an actual time-driven set (render.capture_animation_preview, camera.animation_shots or a level sequence) is not demonstrated. A live editor run with an animation fixture can verify it; this host lacks the Lyra mannequin content (the AGIR fixtures skip as fixture-missing).
