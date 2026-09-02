---
id: B-orbit-shots-no-filename-stem
title: "camera.orbit_shots has no `filename` stem and rounds azimuth to a whole degree in the name it generates, so a set stepped finer than 1 degree cannot be mapped back to poses without parsing the response"
status: OPEN
severity: Low
category: bug
tags: [render, camera, orbit_shots, capture_asset_preview, filename, naming, parity, unknown-params, ergonomics]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# The name is the only thing a downstream tool sees, and it is neither chosen nor exact

Two independent gaps in one format string.

**1. There is no `filename` parameter.** The stem and the output directory are welded:
`PoseRequest.FilenamePrefix = TEXT("CameraOrbit")` and
`PoseRequest.Subdirectory = TEXT("CameraOrbit")` —
`Source/PinWright/Private/Handlers/Render/CameraFrameHandler.cpp:1130-1131`. Nothing in the
verb's `RPC_PARAMS` block (`:680-707`) offers either. Verified live against the running editor,
2026-09-02:

    camera.orbit_shots {subject:{kind:"staticMesh", path:"/Engine/BasicShapes/Cube.Cube"},
                        measureCoverage:true, filename:"probe"}

    [UNKNOWN_PARAMS] Unknown parameter(s) for 'camera.orbit_shots': [measureCoverage, filename].
    Valid parameters: [actorName, objectPath, actorPath, actor_name, point, subject, count,
    angles, elevation, radius, padding, fov, width, height, viewMode, distribution, seed,
    projectionMode, views, exposure, hideEditorSprites, previewScene, inline].

The sibling verb has exactly this parameter, already specified for the multi-shot case:
*"On a multi-shot call (count / views) it is used as the STEM and each shot appends its index and
angles"* (`Handlers/Render/RenderHandler.cpp:319`). So two verbs serving the same asset subject
disagree on whether the caller may name the output. Two consecutive orbit sets of two different
assets land in one directory under one stem, distinguishable only by a wall-clock stamp.

**2. The azimuth in the generated name is rounded to a whole degree.**

    Pose.Filename = FString::Printf(TEXT("CameraOrbit_%s_shot%02d_az%d_el%d.png"),
        *OrbitStamp, ShotIndex,
        FMath::RoundToInt(ShotAzimuth), FMath::RoundToInt(ShotElevation));
    -- Handlers/Render/CameraFrameHandler.cpp:1180-1182

The verb accepts arbitrary `angles[]` (`:687`) and its own `distribution: "ring"` generates
`Azimuth = (360.0f * Index) / Count` (`Handlers/Render/CameraShotPlanUtils.h:564`), which is
non-integral for most counts. A 1.5-degree step therefore names shots `az0, az2, az3, az5, az6,
az8, az9, az11 …` — irregularly spaced tokens, each off the true pose by up to 0.5 degrees. Below
a 1-degree step, distinct poses share an identical `az` token outright.

**No data is lost, and files never collide** — `shot%02d` carries uniqueness within the call
(that is what `B-orbit-shots-filename-collision` fixed), and the full-precision resolved angle is
published per shot as `shots[].angle.azimuth` / `.elevation`
(`Handlers/Render/CameraFrameHandler.cpp:1244-1248`). So the pose is always recoverable — from
the response. That is the whole defect: the filename, which is the only thing an ffmpeg
invocation or an image-diff script ever sees, stops describing the pose, and the mapping has to
be reconstructed by parsing JSON that a shell pipeline does not have.

The rounding was deliberate and its stated reason is sound — the code comment at `:1175-1179`
says az/el are rounded so the name carries no `.`, `/` or `\` that `MakeScreenshotFilename` would
reject, which would fall back to the colliding auto-name. That constraint is a character-set
problem, not an argument for one degree of precision: `az001p50` / `az0015` / `az_1p5` all satisfy
it.

Encountered 2026-09-02 assembling a 240-frame turntable at a 1.5-degree step from ten
`camera.orbit_shots` calls (`F-preview-turntable-capture`): the PNGs on disk could not be ordered
by name, so the sequence had to be rebuilt from `shots[].angle.azimuth` before the mux.

**Fix:** (a) declare `filename` on `camera.orbit_shots` with the semantics
`render.capture_asset_preview` already documents — caller stem, per-shot index and angles
appended; a `subdirectory` alongside it would remove the second half of the collision. (b) Encode
the angle at a precision that survives the plan the verb accepts. Whatever encoding is chosen,
sorting the produced filenames must yield capture order, since that is what makes a name useful
to a tool that cannot read the response.

**Severity Low.** Pure friction by the rubric: naming and discoverability. Nothing is silently
wrong, nothing is lost, and the exact poses are published — the cost is a response parse that a
better name would remove, plus two sets sharing a directory. No reach bump: `camera.orbit_shots`
is not an every-session verb. Explicitly **not** the Medium that
`B-orbit-shots-filename-collision` carried — that one was real data loss (files overwriting each
other); this one is legibility only.

## Related

- `B-orbit-shots-filename-collision` (DONE) — introduced the `_shot%02d_az%d_el%d` format that
  this ticket is about. Its fix is correct and this is not a regression of it: uniqueness holds.
- `B-orbit-shots-no-subject-coverage` — the other half of the same `orbit_shots` /
  `capture_asset_preview` parity gap; independent fixes.
- `F-preview-turntable-capture` — the workflow that surfaced both; a turntable verb needs
  azimuth-ordered names under a caller stem, so it depends on this.
- `B-createpackage-unvalidated-paths-plugin-wide` (IN-REVIEW) — also edits `camera.orbit_shots`'
  parameter declarations; expect a conflict in that block.

## History
- `#1-no-stem-and-rounded-azimuth` `OPEN` reporter — Two gaps in one format string. (1) `camera.orbit_shots` has no `filename` parameter: the stem and subdirectory are welded as `PoseRequest.FilenamePrefix = TEXT("CameraOrbit")` / `Subdirectory = TEXT("CameraOrbit")` (`CameraFrameHandler.cpp:1130-1131`) and neither appears in the `RPC_PARAMS` block (`:680-707`), while the sibling `render.capture_asset_preview` documents exactly this parameter for the multi-shot case ("it is used as the STEM and each shot appends its index and angles", `RenderHandler.cpp:319`). Verified live 2026-09-02: `camera.orbit_shots {subject:{kind:"staticMesh",path:"/Engine/BasicShapes/Cube.Cube"}, measureCoverage:true, filename:"probe"}` returned `[UNKNOWN_PARAMS] ... [measureCoverage, filename]` listing the 23 accepted parameters. (2) The generated name rounds the angles: `CameraOrbit_%s_shot%02d_az%d_el%d.png` with `FMath::RoundToInt(ShotAzimuth)` / `RoundToInt(ShotElevation)` (`CameraFrameHandler.cpp:1180-1182`), while the verb accepts arbitrary `angles[]` (`:687`) and its own ring distribution generates `(360.0f * Index) / Count` (`CameraShotPlanUtils.h:564`), non-integral for most counts. A 1.5-degree step names shots `az0, az2, az3, az5, az6, az8 …` — irregular tokens each up to 0.5 degrees off the pose; below a 1-degree step distinct poses share one token. No data is lost and no file collides: `shot%02d` carries uniqueness (the `B-orbit-shots-filename-collision` fix) and the full-precision angle is published as `shots[].angle.azimuth`/`.elevation` (`CameraFrameHandler.cpp:1244-1248`), so the pose is recoverable — from the response, not from the name an ffmpeg or diff pipeline actually sees. The rounding's stated reason is sound but does not require whole degrees: the comment at `:1175-1179` rounds only to keep `.`, `/` and `\` out of a name `MakeScreenshotFilename` would otherwise reject, which `az001p50` or `az0015` also satisfy. Encountered assembling a 240-frame 1.5-degree turntable from ten calls: the PNGs could not be ordered by name and the sequence was rebuilt from `shots[].angle.azimuth` before the mux.
