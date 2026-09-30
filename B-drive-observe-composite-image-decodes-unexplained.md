---
id: B-drive-observe-composite-image-decodes-unexplained
title: "drive.observe composite screenshot test fails 'image decodes' under host-neutral PIE; assertion gave no size or decode detail"
status: OPEN
severity: Medium
category: bug
tags: [tests, drive, screenshot, pie, diagnostics]
encounters: 1
lastSeen: 2026-09-30T11:37:33Z
---

# drive.observe composite screenshot: 'image decodes' fails, cause not yet observable

`PinWright.drive.observe.ScreenshotIncludesUmgAndMarksAtSurfaceLocalCoords` failed on the full
offscreen suite (`Saved/PinWright/test-runs/693f8d20ebbd4e59979c3e44d76cfdbd/automation.log:27373-27516`)
with `Expected 'image decodes' to be true. [TestDriveObserveScreenshotComposite.cpp(92)]`.
`capture succeeds`, `the on-screen element is marked`, `nothing omitted` and `base64 decodes` all
passed, so `FDriveSetOfMarkRenderer::CaptureAnnotated` returned an encoded frame. The failing
predicate is `bDecoded && W > 220 && H > 220 && Raw.Num() >= W*H*4`; the message did not say which
term failed.

What the logs rule out:
- The host's `LogApp: Error: FRelayClient: Connection error: address resolution failed ...` at PIE
  start did not alter the frame. The passing run
  (`Saved/PinWright/test-runs/integrator_resume2/automation.log:16248-16407`) logged the identical
  error at the same point of PIE start and passed; in the failing run PIE kept ticking normally
  from frame 503 to the capture at ~547 (PixelStreaming join, no ensure, no stall). That error was
  a separate test failure (host issue, fixed host-side by logging it at Warning).
- No `Slate screenshot did not complete in-call` line, so the back-buffer path produced the frame
  (not the scene-only `ReadPixels` fallback, which would also have hit `IsBlankReadback` on the
  unlit Untitled map).

What differs from the last passing runs (2026-09-24): the test now starts PIE through
`FStartHostNeutralPieCommand` (`GameModeOverride = AGameModeBase`) on `/Temp/Untitled_1` instead of
the host's `L_Core` with its GameMode, and the viewport widget's desktop origin moved from
(77.0, 163.8) to (4.0, 133.8). The likeliest remaining explanation is a viewport rect of 220 px or
less on one axis, but no log line records the size.

Related, still open: `B-drive-setofmark-writes-jpeg` (the frame is JPEG while `Mime` says
image/png). The test decodes through `DetectImageFormat`, so JPEG alone does not explain a failure.

**Fix (diagnostic only so far):** the assertion now reads
`image decodes and exceeds the 220x220 mark box (decoded=.. format=.. bytes=.. W=.. H=.. raw=..; reported WxH mime)`,
so the next run names the failing term.

## History
- `#1-unexplained-decode-failure` `OPEN` reporter - Filed from the full offscreen suite run 693f8d20: 'image decodes' failed with no detail. Relay error ruled out with the passing-run comparison above. Assertion message extended in `Tests/Drive/TestDriveObserveScreenshotComposite.cpp` to report decode status, detected format, encoded bytes, decoded W/H, raw size and the renderer's reported size/mime. Root cause needs one live rerun of `PinWright.drive.observe` (offscreen) to read those values.
