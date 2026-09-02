---
id: F-preview-turntable-capture
title: "No verb turns an asset into an N-frame turntable in one call: every preview-scene set is capped at 24 shots, so a 240-frame spin costs 10 calls, 10 preview-scene rig cycles and an out-of-band mux"
status: OPEN
severity: Medium
category: feature
tags: [render, camera, orbit_shots, capture_asset_preview, turntable, preview-scene, shot-cap, video, showcase]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# One asset, 360 degrees, one call

Showing a generated mesh off is the plugin's own use case, and the shape it wants is a turntable:
N frames around one static asset in its preview scene, constant size, one pinned exposure, one
explicit rig, one output. Today that is 10 calls plus an out-of-band mux.

Measured 2026-09-02 while producing a showcase video from `/Game/Maps/Atlantis`: a 240-frame
turntable of one column took **10 `camera.orbit_shots` calls of 24 explicit `angles` each**, plus
ffmpeg to assemble the PNGs, plus a reorder step because the ten calls' outputs interleave on
disk. The per-call preview-scene rig was applied and restored **10 times** for one continuous spin.

## Why it takes ten calls

**The 24-shot ceiling is the whole blocker, and it is family-wide.**
`GMaxOrbitShots = 24` — `Source/PinWright/Private/Handlers/Render/CameraShotPlanUtils.h:60`.
`camera.orbit_shots` refuses a longer plan with `TOO_MANY_SHOTS` before any capture runs
(`Handlers/Render/CameraFrameHandler.cpp:927-931`) and forwards the same bound into the shared
primitive as `PoseRequest.MaxPoses` (`:1141`), which truncates rather than refuses
(`Handlers/Render/PoseListCapture.cpp:65-66`). `render.capture_asset_preview` is under the same
ceiling — its own `times` parameter says so: "the combined total is bounded by the same 24-shot
ceiling as count/views" (`Handlers/Render/RenderHandler.cpp:349`). Documented for callers at
`Saved/PinWright/wiki/camera.orbit_shots.md:103`.

**A fixed angular step is already expressible — the cap is what stops it.** Correcting the
report this ticket was filed from: `distribution: "ring"` produces `Azimuth = (360.0f * Index) /
Count` (`Handlers/Render/CameraShotPlanUtils.h:564`), which is an exactly constant step of
`360/Count` degrees. `count: 240, distribution: "ring"` **is** the 1.5-degree turntable; it is
rejected only by the ceiling. So this ticket is not asking for a new pose generator. It is asking
for a set longer than 24 and for the two things a caller then has to do by hand.

**Neither verb alone can serve the workflow.**

| | `camera.orbit_shots` | `render.capture_asset_preview` |
|---|---|---|
| explicit `angles[]` | yes (`CameraFrameHandler.cpp:687`) | **no** — `count` / `views` / `times` only (`RenderHandler.cpp:343-344,349`) |
| `filename` stem | **no** (`B-orbit-shots-no-filename-stem`) | yes, used as a stem on multi-shot calls (`RenderHandler.cpp:319`) |
| `measureCoverage` | **no** (`B-orbit-shots-no-subject-coverage`) | yes, default true (`RenderHandler.cpp:339`) |
| max shots per call | 24 | 24 |
| video output | none | none |

Verified live against the running editor, 2026-09-02:
`camera.orbit_shots {subject:{kind:"staticMesh",path:"/Engine/BasicShapes/Cube.Cube"},
measureCoverage:true, filename:"probe"}` →
`[UNKNOWN_PARAMS] Unknown parameter(s) for 'camera.orbit_shots': [measureCoverage, filename].
Valid parameters: [actorName, objectPath, actorPath, actor_name, point, subject, count, angles,
elevation, radius, padding, fov, width, height, viewMode, distribution, seed, projectionMode,
views, exposure, hideEditorSprites, previewScene, inline].`

**Nothing in the plugin muxes frames.** Every capture verb writes one PNG per shot. The only
video-producing namespace is `mrq.*`, and it renders a **Level Sequence in a map**
(`mrq.create_job` takes `sequencePath` + `levelPath`, `Handlers/MRQ/MRQHandler.cpp:205-217`) — so
using it for an asset turntable means first authoring a camera-orbit sequence and a level
containing the asset, which is the many-step workflow this ticket exists to remove.

**Ten calls means ten rig cycles.** The `previewScene` rig is applied once and restored once per
*call*, by design ("Read once for the whole set... and restored once after the last one",
`CameraFrameHandler.cpp:706`; the restore itself at `Handlers/Render/PreviewSceneRig.cpp:703,708`
and `:787-788`). Correct for one set; ten times over for one spin, and each restore round-trips
the process-wide `UAssetViewerSettings` profile array and the committed
`Config/DefaultEditor.ini`.

## Ask

One verb — `render.capture_turntable`, or a `turntable` plan on `render.capture_asset_preview` —
taking a subject, a frame count (or an explicit angular step), an elevation, a size, an exposure
pin and a `previewScene` rig, and producing **one output**: a numbered PNG sequence under a
caller-named stem, or an MP4. Requirements that follow from what already exists:

- **Frames land in azimuth order under a caller-supplied stem**, zero-padded, so a mux needs no
  reordering and no response parsing. Both halves are separately filed as
  `B-orbit-shots-no-filename-stem`.
- **One rig application for the whole spin**, not one per batch.
- **The existing per-frame proof is kept.** `framing` / `boundsInFrame` already rides on the
  shared primitive whenever bounds are supplied (`PoseListCapture.h:231-240`), and
  `subjectCoverage` is the only signal that catches a frame containing nothing
  (`RenderHandler.cpp:339`) — a 240-frame set is exactly where nobody will eyeball every frame,
  so the differential matters more here, not less. See `B-orbit-shots-no-subject-coverage`.
- **The 24 ceiling has to be argued, not inherited.** It is not a product limit: the only
  evidence behind it on this board is `F-animated-capture-verbs`, which cites 24 shots as the
  longest burst validated against the `FViewport::GetHitProxy` assert class that had been killing
  the editor. Whoever raises it owns re-validating that class at the new length, or the verb
  batches internally under the proven bound while staying one call to the caller.

**Severity Medium.** Soft blocker by the board rubric: the output is reachable, but only through
many extra calls plus out-of-band tooling — 10 calls, an ffmpeg step and a reorder for what is one
request. Not High: nothing is silently wrong, and no valid input is rejected except by a bound the
verb states honestly (`TOO_MANY_SHOTS`). No reach bump — showcase capture is not an
every-session path.

## Related

- `B-orbit-shots-no-subject-coverage` — same verb, the coverage half of the parity gap.
- `B-orbit-shots-no-filename-stem` — same verb, the naming half; a turntable needs both.
- `B-capture-preview-decoration-not-suppressible` — a turntable frame still carries the preview
  viewport's axis gizmo and grid, with no lever for either. A turntable verb that ships before
  that one is fixed produces 240 frames that all need cropping.
- `F-animated-capture-verbs` (DONE) — the architectural precedent to extend, not a duplicate: its
  two verbs cross **instants** with a small angle plan (`frameCount` capped at 5 per view), which
  is the time axis. A turntable is one instant across many azimuths.
- `F-mrq-render-queue` (DONE) — the only video path, and it needs an authored sequence and a map.

## History
- `#1-ten-calls-for-one-spin` `OPEN` reporter — A 240-frame turntable of one static mesh from the preview scene took 10 `camera.orbit_shots` calls of 24 explicit `angles` each, plus an ffmpeg mux and a reorder pass, plus 10 preview-scene rig apply/restore cycles for one continuous spin (measured 2026-09-02 producing a showcase video from `/Game/Maps/Atlantis`). The single blocker is the 24-shot ceiling: `GMaxOrbitShots = 24` (`CameraShotPlanUtils.h:60`), refused with `TOO_MANY_SHOTS` at `CameraFrameHandler.cpp:927-931` and forwarded as `PoseRequest.MaxPoses` at `:1141` where it truncates instead (`PoseListCapture.cpp:65-66`); `render.capture_asset_preview` shares it ("the combined total is bounded by the same 24-shot ceiling as count/views", `RenderHandler.cpp:349`). **Correcting the report:** a fixed angular step IS already expressible — `distribution:"ring"` gives `Azimuth = (360.0f * Index) / Count` (`CameraShotPlanUtils.h:564`), an exactly constant `360/Count` step, so `count:240` is the 1.5-degree turntable and only the ceiling rejects it. Neither verb serves the workflow alone: `orbit_shots` has explicit `angles[]` (`CameraFrameHandler.cpp:687`) but no `filename` stem and no `measureCoverage`; `capture_asset_preview` has both but no `angles[]`. Verified live: `camera.orbit_shots {subject:{kind:"staticMesh",path:"/Engine/BasicShapes/Cube.Cube"}, measureCoverage:true, filename:"probe"}` returns `UNKNOWN_PARAMS` naming both fields and lists the 23 accepted parameters. No verb muxes frames — `mrq.*` is the only video path and it renders a Level Sequence in a map (`MRQHandler.cpp:205-217`), so an asset turntable through it needs a hand-authored sequence and level first. The `previewScene` rig is per-call by design (`CameraFrameHandler.cpp:706`, restore at `PreviewSceneRig.cpp:703,708,787-788`), so ten calls round-trip the process-wide `UAssetViewerSettings` array and `Config/DefaultEditor.ini` ten times. Ask: one verb, N frames over 360 degrees, constant size, pinned exposure, explicit rig, azimuth-ordered output under a caller-named stem, per-frame coverage proof kept. The 24 bound must be re-argued rather than inherited — the only evidence behind it is `F-animated-capture-verbs`' note that 24 shots was the longest burst validated against the `FViewport::GetHitProxy` assert class.
