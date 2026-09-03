---
id: B-orbit-shots-filename-collision
title: "camera.orbit_shots overwrites shots that share a timestamp filename"
status: DONE
severity: Medium
category: bug
tags: [camera, screenshot, render, filename-collision, data-loss]
---

# camera.orbit_shots overwrites shots that share a timestamp filename

`camera.orbit_shots` captures N shots in a loop. Pre-fix each shot left
`Request.Filename` empty, so the capture util auto-names the PNG from a
wall-clock timestamp with only SECOND resolution — `MakeScreenshotFilename`
in `Utils/ScreenshotUtils.cpp` (lines 19-45) uses `%Y%m%d_%H%M%S` when the
filename is empty, and `PreviewViewportCaptureUtils.cpp:84` fills the request
filename from the (empty) `filename` payload field. An orbit set completes
well under a second, so multiple shots get the SAME filename and OVERWRITE
each other on disk. The response still lists all N shots with distinct-looking
metadata (differing sizeBytes), but on disk only the shots that fell in
different seconds survive: a 4-shot orbit can leave only 2 files while 3 of the
response's `path` values are identical. Silent data loss plus a response that
lies about which files are actually present.

Observed: `call("camera.orbit_shots", {actorName:"SmokeCube"})` returned 4
shots; shots 1-3 all reported `path:
.../Saved/Screenshots/CameraOrbit/CameraOrbit_20260704_131849.png` (identical)
and shot 4 `..._131850.png`. Only 2 distinct files on disk.

Severity Medium: the loss is regenerable screenshot output artifacts, not
durable asset data (so not Critical), and the verb is a rare specialized path
(no reach bump). Impact is soft loss plus a response whose per-shot paths do
not match disk.

**Workaround:** space captures more than one second apart (one shot per second)
so each auto-name lands in a distinct wall-clock second.
**Fix:** each shot now sets `Request.Filename =
CameraOrbit_<stamp>_shot<NN>_az<>_el<>.png` in the orbit loop
(`Handlers/Render/CameraFrameHandler.cpp:481,493-500`); the per-shot index
guarantees uniqueness within the call.

## History
- `#1-timestamp-collision-repro` `OPEN` reporter — `camera.orbit_shots` left each shot's `Request.Filename` empty, so `MakeScreenshotFilename` (`Utils/ScreenshotUtils.cpp:19-45`) auto-named from a SECOND-resolution timestamp (`%Y%m%d_%H%M%S`). Shots in the same second collided on one filename and overwrote each other on disk while the response still listed every shot. Repro: `call("camera.orbit_shots", {actorName:"SmokeCube"})` returned 4 shots but shots 1-3 shared `path` `.../CameraOrbit/CameraOrbit_20260704_131849.png` — only 2 distinct PNGs survived on disk.
- `#2-per-shot-indexed-filename` `IN-REVIEW` developer — In `Handlers/Render/CameraFrameHandler.cpp` the orbit loop now computes one shared `OrbitStamp` (line 481) and sets a per-shot `Request.Filename = CameraOrbit_<stamp>_shot%02d_az%d_el%d.png` (lines 493-500); the `shotNN` index guarantees uniqueness, az/el keep the name self-describing. Regression test `PinWright.camera.orbit_shots.DistinctShotPaths` in `Tests/Render/TestCameraFrameHandlers.cpp` (lines 516-517) asserts distinct-path-count equals shot-count (assertion at lines 582-583); PASSES under CLI automation. Uncommitted.

- `#3-verified-six-in-one-second` `DONE` tester — Verified incidentally but decisively while
  runtime-testing the new `views:"sides"` plan (separate workstream). A single
  `camera.orbit_shots {actorName, views:"sides"}` produced SIX shots that all landed in the same
  wall-clock second — the exact condition that used to collapse them — and all six survived on disk
  with distinct names and distinct sizes:
  `CameraOrbit_20260813_143913_shot00_az0_el0.png` (69138 B), `shot01_az180_el0` (104402 B),
  `shot02_az90_el0` (68939 B), `shot03_az-90_el0` (84631 B), `shot04_az0_el90` (435067 B),
  `shot05_az0_el-90` (24237 B). Pre-fix this set would have left ONE file with five of the six
  response paths pointing at it. The response's per-shot `path` values match disk one-for-one, so the
  "response lies about which files are present" half is fixed too. Content was checked as well as
  filenames: all six decode as valid PNGs, and pairwise comparison over a 64x64 luma grid found
  0 near-duplicate pairs out of 15 (minimum 49.2% of cells differing), so they are six genuinely
  different images rather than six copies under six names.
  Correction to `#2`: its closing "Uncommitted" is stale — the fix was already committed in
  `b0e1a6d6`, so this verification is of shipped code, and no later commit re-bundled it.
