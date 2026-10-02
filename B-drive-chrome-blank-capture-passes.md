---
id: B-drive-chrome-blank-capture-passes
title: "drive.observe editor-chrome capture stamps alpha opaque before the blank-readback check, so an all-zero window capture passes as a real screenshot"
status: DONE
severity: Medium
category: bug
tags: [drive, drive-observe, editor-chrome, capture, blank-capture, set-of-mark, false-success, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
---

# Editor-chrome Set-of-Mark capture defeats its own blank check

`PinWrightScreenshotUtils::IsBlankReadback` (`Utils/ScreenshotUtils.h:108-122`) detects a surface
that was never drawn by testing that every pixel is zero in all four channels, and its contract says
it MUST run before `ForceOpaqueAlpha`, because the alpha stamp turns an all-zero readback into a
plausible opaque black frame.

`FDriveSetOfMarkRenderer` honours that order for the game and web surfaces
(`Handlers/Drive/DriveSetOfMarkRenderer.cpp:175-191`: `IsBlankReadback` -> `BLANK_CAPTURE`, then
`ForceOpaqueAlpha`). The editor-chrome surface does not: it gets its pixels from
`FDriveEditorChrome::CaptureWindow` (`DriveSetOfMarkRenderer.cpp:108`), and `CaptureWindow` already
calls `ForceOpaqueAlpha(Bitmap)` before returning (`Handlers/Drive/DriveEditorChrome.cpp:596-597`).
By the time the shared check at `:179` runs, every alpha byte is 0xFF, so `IsBlankReadback` can
never return true on that surface. A window the renderer never drew (minimized, not presented in an
offscreen run, just created) comes back as a black frame with Set-of-Mark badges painted over it and
`success:true`, which is the exact false observation the check exists to prevent (see the comment at
`DriveSetOfMarkRenderer.cpp:174-178`).

Verified by source read; not reproduced live. `CaptureWindow` has one caller (the renderer above), so
the stamp inside it is redundant: the renderer stamps alpha itself at `:191` after the check.

**Impact:** silent false-success on drive.observe / action verbs targeting editor chrome when the
window was not drawn (High impact class), on a rarer path than the game viewport (bumped down one).
**Fix:** delete the `ForceOpaqueAlpha` call and its comment from `FDriveEditorChrome::CaptureWindow`,
so the shared blank check sees the raw readback. Add a test that feeds an all-zero bitmap through the
editor-chrome path (or asserts `CaptureWindow` returns alpha untouched) and expects `BLANK_CAPTURE`.

## History
- `#1-alpha-stamp-before-blank-check` `OPEN` reporter — Found in today's verification: `DriveEditorChrome.cpp:597` stamps opaque alpha before `DriveSetOfMarkRenderer.cpp:179` checks for an all-zero readback, so the check is dead on the editor-chrome surface. Source-verified; only caller of `CaptureWindow` is the Set-of-Mark renderer.
- `#2-chrome-stamp-removed` `IN-REVIEW` developer — Deleted the `ForceOpaqueAlpha` call from `FDriveEditorChrome::CaptureWindow` (`Handlers/Drive/DriveEditorChrome.cpp`), so the renderer's shared `IsBlankReadback` check in `FDriveSetOfMarkRenderer::CaptureAnnotated` sees the raw readback on the editor-chrome surface and the renderer's own stamp (after the check) still makes the encoded frame opaque. Header contract comment in `DriveEditorChrome.h` and the cross-reference in `Handlers/UI/WidgetDesignerScreenshotHandler.cpp` updated; `docs/wiki-src/drive.md` (drive.observe `BLANK_CAPTURE` paragraph) now covers `editor_chrome`; CHANGELOG entry. Test `PinWright.drive.editorchrome.CaptureLeavesAlphaForBlankCheck` (`Tests/Drive/TestDriveEditorChromeBlankCapture.cpp`) is a source contract: `CaptureWindow`'s comment-stripped body must not call `ForceOpaqueAlpha`, and in `CaptureAnnotated` the chrome capture, `IsBlankReadback(` and `ForceOpaqueAlpha(` must appear in that order. Fails if the stamp is restored. Not a live capture because no headless configuration makes Slate return an all-zero window readback on demand (an undrawn window fails `TakeScreenshot` with `CAPTURE_FAILED` instead). Compile-checked with UBT -SingleFile.
- `#3-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.drive.editorchrome.CaptureLeavesAlphaForBlankCheck` passed in w23-final (no skip), with the other five `drive.editorchrome.*` tests. The asked fix is met: the test asserts the comment-stripped `FDriveEditorChrome::CaptureWindow` body no longer calls `ForceOpaqueAlpha`, and that `CaptureAnnotated` orders chrome capture -> `IsBlankReadback(` -> `ForceOpaqueAlpha(`. Limit: it is a source-text contract, not a runtime capture of an all-zero window, so `BLANK_CAPTURE` on the chrome surface is not observed end to end; the ticket allowed either test form.
