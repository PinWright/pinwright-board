---
id: B-subject-coverage-blind-to-shadowed-subject
title: "subjectCoverage reads ~0 for a subject that IS drawn, when every visible face is in full shadow against a black show-only background: the measure is a per-channel frame difference with a fixed threshold of 8, so black-on-black pixels never count"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_actor_preview, capture_mesh, subjectCoverage, measureCoverage, frame-difference, threshold, shadow, show-only, black-background, false-empty]
encounters: 1
lastSeen: 2026-10-02T18:00:00-05:00
rice: [1, 2, 1, 1]
priority: 17
---

# A drawn subject in shadow scores the same as an empty frame

## What happens

`render.capture_actor_preview` (`Handlers/Render/ActorPreviewCapture.cpp:129-146`) and
`render.capture_mesh` (`Handlers/Render/MeshPreviewCaptureUtils.cpp:331-349`) compute
`subjectCoverage` as `PinWrightFlatRegion::MeasureFrameDifference(Reference, Frame, 8).ChangedPixelFraction`:
the fraction of pixels where any channel differs by more than 8/255 between the frame and a
reference drawn without the subject. `capture_actor_preview` draws show-only, so its reference
frame is black. A subject whose camera-facing faces are all unlit renders near-black too, the
difference stays under 8, and coverage reads ~0. Nothing in the response says "the subject was
drawn but could not be told from the background"; it reads exactly like an empty frame.

## Measured

Found while fixing `PinWright.render.capture_actor_preview.ExistingActorDrawnShowOnlyAndUntouched`
(batch-4 G20, run2 diagnostics `g20-diag1..3`). The test's `FPreviewScene` key light arrives from
azimuth 112.5; the capture camera sat at azimuth 0 (+X looking -X). The subject's proxy existed and
was registered before the capture (diag3), yet the frame had mean luminance 0.008, max 0.047, and
`subjectCoverage` 0. The same capture at azimuth 112.5 gave coverage 0.598. The test was fixed by
re-aiming its key light (`SetLightRotation(-40,180,0)`); the verbs were not changed. See
`F-render-runtime-spawned-actor` `#5-existing-actor-test-lighting`.

## Why it matters

The parameter docs sell `subjectCoverage` as the one signal that catches a frame with nothing in it.
For these two verbs it also fires on a frame that holds the subject unlit, so a caller who trusts it
concludes "subject missing" and goes looking for a resolve, bounds or visibility bug that does not
exist (the G20 fixer did exactly that for one round). `capture_asset_preview` is less exposed
because its reference frame keeps the lit floor and sky backdrop, so a dark subject still differs
from it.

## What should happen

Pick one, do not guess both:
- measure coverage from something that does not depend on shading (a depth or custom-stencil /
  primitive-ID pass of the same pose, or a second reference drawn against a non-black clear colour),
  or
- keep the luminance measure but say so: when coverage is ~0 and the subject's projected bounds are
  in frame, publish a `coverageWarning` that names "drawn unlit against a black background" as a
  possible cause and points at the light direction / `azimuth`.

## History
- `#1-filed-from-g20-run2` `OPEN` reporter — Filed from the batch-4 G20 run2 diagnosis (`b5/fix-G20.md`, "run2 fix" side note). Source cited above; repro is the pre-fix `ExistingActorDrawnShowOnlyAndUntouched` setup (key light from azimuth 112.5, camera at azimuth 0, show-only). No production change made.
