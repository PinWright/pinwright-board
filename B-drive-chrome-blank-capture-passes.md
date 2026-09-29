---
id: B-drive-chrome-blank-capture-passes
title: "drive.observe editor-chrome capture stamps alpha opaque before the blank-readback check, so an all-zero window capture passes as a real screenshot"
status: OPEN
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
