---
id: B-orbit-shots-no-subject-coverage
title: "camera.orbit_shots serves the same asset subject as render.capture_asset_preview through the same capture primitive, but cannot ask for subjectCoverage — the one signal that catches a frame containing nothing"
status: OPEN
severity: Medium
category: bug
tags: [render, camera, orbit_shots, capture_asset_preview, measureCoverage, subject, parity, pass-through-gap, unknown-params]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# Same subject, same primitive, one verb gets the measurement

`render.capture_asset_preview` and `camera.orbit_shots` both take `subject: {kind, path}`, both
open that asset's editor preview, and both run their poses through the *same* primitive
`PinWrightPoseCapture::CaptureCameraPoses`. One of them can measure whether the subject is
actually in the picture. The other cannot ask.

**Declared on one verb only.** `measureCoverage` is an `RPC_PARAM_DEF(..., "true")` on
`render.capture_asset_preview` — `Source/PinWright/Private/Handlers/Render/RenderHandler.cpp:339`
— and appears nowhere in `camera.orbit_shots`' `RPC_PARAMS` block
(`Handlers/Render/CameraFrameHandler.cpp:680-707`). The dispatcher's strict unknown-param gate
therefore refuses it. Verified live against the running editor, 2026-09-02:

    camera.orbit_shots {subject:{kind:"staticMesh", path:"/Engine/BasicShapes/Cube.Cube"},
                        measureCoverage:true, filename:"probe"}

    [UNKNOWN_PARAMS] Unknown parameter(s) for 'camera.orbit_shots': [measureCoverage, filename].
    Valid parameters: [actorName, objectPath, actorPath, actor_name, point, subject, count,
    angles, elevation, radius, padding, fov, width, height, viewMode, distribution, seed,
    projectionMode, views, exposure, hideEditorSprites, previewScene, inline].

**It is a pass-through gap, not a missing capability.** Everything the differential needs already
resolves on the orbit path:

- The primitive gates on `Request.bMeasureSubjectCoverage && Request.SubjectVisibilitySetter`
  (`Handlers/Render/PoseListCapture.cpp:74-76`). `camera.orbit_shots` sets **neither**: its
  `FPoseListCaptureRequest` fill (`Handlers/Render/CameraFrameHandler.cpp:1128-1149`) assigns
  `Width`, `Height`, `FilenamePrefix`, `Subdirectory`, `Exposure`, `bHideEditorSprites`,
  `ViewMode`, `PreviewSceneRig`, `MaxPoses`, `BoundsOrigin`, `BoundsRadius` — and no coverage
  field. Both default false/unbound (`Handlers/Render/PoseListCapture.h:203,216`).
- The visibility setter it would forward already exists on the resolved subject as
  `FResolvedSubject::VisibilitySetter` (`Handlers/Render/CaptureSubject.h:224`), which
  `render.capture_asset_preview` forwards in one line
  (`Handlers/Render/RenderHandler.cpp:1197: PoseRequest.SubjectVisibilitySetter =
  Resolved.VisibilitySetter;`) alongside the wire read at `:1196`.

So the fix is two lines on a verb that already resolves the subject through the same provider
registry — not new rendering.

## Why this costs more than a missing knob

`measureCoverage`'s own documentation states what is lost
(`Handlers/Render/RenderHandler.cpp:339`): it is *"the only published signal that catches a frame
containing NOTHING: over pure backdrop `boundsInFrame`, `blank` and `litPixelFraction` all read
healthy, because geometrically the bounds are still in frame and the backdrop really is lit."*

`camera.orbit_shots` still emits the `framing` / `boundsInFrame` verdict (bounds are supplied at
`CameraFrameHandler.cpp:1145-1146`), so an orbit set over an empty preview scene returns N shots
that all read healthy on every field the verb publishes. And because coverage was never
requested, `coverageReferenceShots` is suppressed too — it is emitted only under
`if (Result.bCoverageRequested)` (`Handlers/Render/PoseListCapture.cpp:452-457`) — so nothing in
the response distinguishes "measured and fine" from "nobody could have measured this".

**And `camera.orbit_shots` is the verb you are forced onto for the sets that need it most.** It
is the only one of the two that accepts an explicit `angles[]` list
(`CameraFrameHandler.cpp:687`); `render.capture_asset_preview` offers `count` / `views` / `times`
only (`RenderHandler.cpp:343-344,349`). So any set with hand-chosen poses — the long ones nobody
eyeballs frame by frame — is exactly the set with no coverage proof available. Encountered
2026-09-02 building a 240-frame turntable for a showcase video (see
`F-preview-turntable-capture`).

## What is NOT the defect

The doubled draw cost is documented, deliberate and already opt-out on the verb that has the
knob: *"Costs one extra draw plus readback per shot — roughly double the capture time for a set
— which is the reason it can be turned off"* (`RenderHandler.cpp:339`). Defaulting the primitive
to `false` is likewise deliberate and correct (`PoseListCapture.h:211-216`: a verb that has not
thought about the differential must not start paying for it silently). The defect is that
`camera.orbit_shots` has no way to opt **in**.

**Fix:** declare `measureCoverage` on `camera.orbit_shots` and forward both fields the way
`RenderHandler.cpp:1196-1197` already does. Two open questions for whoever takes it: (a) whether
the default should be `true` for parity or `false` because this verb also serves `actorName` and
bare-`point` targets, which have no preview component to hide — the primitive already answers an
unmeasurable request with an absent number rather than an error
(`PoseListCapture.cpp:71-76`), so `true` is defensible; (b) whether the same declaration belongs
on `camera.animation_shots`, which shares the primitive.

**Severity Medium.** Soft blocker by the rubric: the measurement is reachable only by re-shooting
the whole set through a verb that cannot express the poses you asked for, which is the
"documented workaround / many extra calls" band. Not High — nothing is silently wrong on a
correct capture, and the absence is not dressed up as a measurement (`subjectCoverage: 0` is
deliberately never emitted). Not Low — this is a real signal you cannot obtain, not friction.
No reach bump: neither verb runs in most sessions.

## Related

- `B-orbit-shots-no-filename-stem` — the other half of the same `orbit_shots` /
  `capture_asset_preview` parity gap, filed separately because the fixes are independent.
- `F-preview-turntable-capture` — the workflow that surfaced both.
- `B-verbs-read-undeclared-parameters` (IN-REVIEW) — the inverse shape (handler reads a param its
  `RPC_PARAMS` never declares); `camera.orbit_shots` is not in its table, and here the field is
  neither declared nor read.
- `B-capture-preview-decoration-not-suppressible` (OPEN) — the same pass-through-gap shape on the
  same preview viewport.

## History
- `#1-coverage-unreachable-from-orbit` `OPEN` reporter — `measureCoverage` is declared only on `render.capture_asset_preview` (`RenderHandler.cpp:339`, `RPC_PARAM_DEF` default `"true"`) and is absent from `camera.orbit_shots`' `RPC_PARAMS` block (`CameraFrameHandler.cpp:680-707`), so the dispatcher's strict gate refuses it. Verified live 2026-09-02: `camera.orbit_shots {subject:{kind:"staticMesh",path:"/Engine/BasicShapes/Cube.Cube"}, measureCoverage:true, filename:"probe"}` returned `[UNKNOWN_PARAMS] ... [measureCoverage, filename]` and listed the 23 accepted parameters. Both verbs take the same `subject:{kind,path}`, open the same asset-editor preview and run through the same primitive `PinWrightPoseCapture::CaptureCameraPoses`, which gates the differential on `Request.bMeasureSubjectCoverage && Request.SubjectVisibilitySetter` (`PoseListCapture.cpp:74-76`). `camera.orbit_shots` sets neither in its request fill (`CameraFrameHandler.cpp:1128-1149`), and both default off (`PoseListCapture.h:203,216`), while `render.capture_asset_preview` forwards them in two lines (`RenderHandler.cpp:1196-1197`) off a setter that already exists on the resolved subject (`CaptureSubject.h:224`) — a pass-through gap, not a missing capability. The cost: `measureCoverage` is documented as "the only published signal that catches a frame containing NOTHING: over pure backdrop `boundsInFrame`, `blank` and `litPixelFraction` all read healthy" (`RenderHandler.cpp:339`), and `camera.orbit_shots` does emit `framing`/`boundsInFrame` (bounds set at `CameraFrameHandler.cpp:1145-1146`), so an orbit set over an empty preview scene reads healthy on every field it publishes; `coverageReferenceShots` is also suppressed because it is emitted only under `if (Result.bCoverageRequested)` (`PoseListCapture.cpp:452-457`). Worse, `camera.orbit_shots` is the only one of the two accepting an explicit `angles[]` list (`CameraFrameHandler.cpp:687`; the other offers `count`/`views`/`times` at `RenderHandler.cpp:343-344,349`), so the long hand-posed sets nobody eyeballs frame by frame are exactly the ones with no coverage proof available. Encountered building a 240-frame turntable for a showcase video from `/Game/Maps/Atlantis`. NOT the defect: the doubled draw cost, which is documented, deliberate and already opt-out where the knob exists, and the primitive's `false` default, which is deliberate (`PoseListCapture.h:211-216`).
