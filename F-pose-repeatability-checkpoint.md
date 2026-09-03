---
id: F-pose-repeatability-checkpoint
title: "Time-driven pose captures cannot measure repeatability because no subject-state checkpoint can replay pose zero"
status: OPEN
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
