---
id: B-drive-observe-composite-image-decodes-unexplained
title: "drive.observe composite screenshot test fails 'image decodes' under host-neutral PIE; assertion gave no size or decode detail"
status: IN-REVIEW
severity: Medium
category: bug
tags: [tests, drive, screenshot, pie, diagnostics]
encounters: 4
lastSeen: 2026-10-01T13:04:08Z
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
- `#2-decode-not-recurring-relay-only` `IN-REVIEW` developer — Three offscreen runs on 2026-10-01 (`Saved/PinWright/test-runs/b2aea516a7ed4e7ea30dc6e14c201f76`, `e65422727528455780ea7a4d92afc515`, `dc867ac2ec5642e087cc2dd40e386ad1`, host wt2 checkout) failed this test, and in every one the only error event was the host's `LogApp: Error: FRelayClient: Connection error: address resolution failed ...`; the extended 'image decodes and exceeds the 220x220 mark box' assertion passed all three times (viewport origin (77.0, 163.8)). `PinWright.editor.simulate_input.PiePlayerInputDelivery` failed on the same line in two of the three runs and passed in the third, because the relay error is logged when an async DNS lookup completes and so lands inside or after whichever PIE test is running. That error is host-project code, not plugin code: the host fixed it on its master as commit 6557fb7e2e (logs it at Warning); the wt2 checkout was behind that commit, so the identical one-line change was applied to `Plugins/App/Source/App/Networking/RelayClient.cpp` in the wt2 working tree. No plugin-side allow-list was added: the existing mechanism (`ExpectHostPieStartErrors` in `Tests/Drive/HostNeutralPie.h`) is reserved for engine-shipped plugins with a verified gate, and putting a host's log string in the plugin would violate the plugin's general-purpose rule. The decode/size failure from #1 has not recurred since the assertion started naming its terms; if it does, the message now names the failing term. Note for a reviewer: when the viewport's desktop x is <= 7, the mark-outline counterfactual (b) stops discriminating, because the old desktop-space outline at 100+x falls inside the 97..104 search window. Verify: rerun `PinWright.drive.observe.ScreenshotIncludesUmgAndMarksAtSurfaceLocalCoords` and `PinWright.editor.simulate_input.PiePlayerInputDelivery` offscreen after rebuilding the host App module.
