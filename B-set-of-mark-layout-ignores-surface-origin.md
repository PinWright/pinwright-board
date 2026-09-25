---
id: B-set-of-mark-layout-ignores-surface-origin
title: "Set-of-Mark marks are painted at desktop coordinates onto a viewport-local / window-local bitmap — `FDriveSetOfMarkLayout::BuildLayout` gets no surface origin, so every mark on `drive.observe`'s screenshot is offset by the captured surface's desktop position (the +37,+254 the geometry ticket measured), and off-frame elements are silently reported as `marks_omitted`"
status: IN-REVIEW
severity: High
category: bug
tags: [drive, drive.observe, set-of-mark, screenshot, geometry, coordinate-space, desktop-space, viewport-origin, marks-omitted, silent-wrong-data]
encounters: 2
lastSeen: 2026-09-23T20:55:00Z
---

# The marks are in desktop space; the image they are drawn on is not

`drive.observe` returns a Set-of-Mark screenshot: numbered boxes painted over the
interactables so an agent can name a target by its number. The boxes are placed from each
element's `geometry.absolute`, which is **desktop** space (the owning window's screen
origin is included — see `B-drive-geometry-window-space-injected-as-desktop`). The bitmap
they are painted onto is not desktop space on either surface.

## Cause: a layout that assumes the frame starts at the desktop origin

`Source/PinWright/Private/Handlers/Drive/DriveSetOfMarkLayout.cpp:16-34`

```cpp
const double FrameMaxX = static_cast<double>(FrameWidth);
const double FrameMaxY = static_cast<double>(FrameHeight);
...
const FVector2D ElemMin = Element.AbsolutePosition;
const FVector2D ElemMax = Element.AbsolutePosition + Element.AbsoluteSize;
```

`BuildLayout(Elements, FrameWidth, FrameHeight, MarkCap)` takes a frame **size** and no
frame **origin**, so it treats the frame as the rect `[0,0]..[Width,Height]` in the same
space as `Element.AbsolutePosition`. That is only true when the captured surface's
top-left is at desktop (0,0).

The caller passes exactly that, for both surfaces:
`Source/PinWright/Private/Handlers/Drive/DriveSetOfMarkRenderer.cpp:172-176` calls
`BuildLayout(Interactables, Width, Height, EffectiveCap)` with the `Width`/`Height` of
whichever bitmap was just captured, and nothing else.

Neither capture is rooted at the desktop origin:

- **Game / web** (`DriveSetOfMarkRenderer.cpp:110-134`) — `FViewport::ReadPixels` +
  `FViewport::GetSizeXY()`. The bitmap's (0,0) is the **game viewport's** top-left. With
  PIE hosted in the level-editor viewport, that sits well inside the editor frame:
  `B-drive-geometry-window-space-injected-as-desktop` `#1` measured that offset as
  **(+37,+254)** on a maximized editor at desktop (0,0). So the game surface — the default
  surface, and the common case — is wrong **even when the editor is maximized**; there is
  no "normal layout makes the missing term zero" escape here, unlike the injection path.
- **Editor chrome** (`DriveSetOfMarkRenderer.cpp:102-108`) — `FDriveEditorChrome::CaptureWindow`
  renders one window, so the bitmap's (0,0) is that **window's** client top-left. Correct
  only while that window is at desktop (0,0).

## Symptom, and why it is silent

Every mark is displaced by the surface origin. Two ways that reaches the caller as wrong
data rather than as an obvious defect:

- A mark lands on the wrong widget. The agent reads the number off the image, targets that
  number's handle, and acts on something else — or, worse, reads the image to decide
  *which* control to press and picks the one the misplaced box happens to cover.
- An element whose desktop rect falls outside `[0,0]..[Width,Height]` is dropped by the
  `bOverlapsFrame` test (`DriveSetOfMarkLayout.cpp:38-45`) and reported in
  `marks_omitted` — which the response documents as "offscreen, too small, or past the
  cap". So a perfectly on-screen control is reported as offscreen, honestly-looking, with
  no error.

The two failure modes compound: the frame is shifted, so elements near the bottom/right of
the surface fall off the far edge while elements near the top/left get boxes drawn over
whatever is 37 px left and 254 px up from them.

## Fix

`BuildLayout` needs the captured frame's **origin** in the same space as the elements, and
must subtract it before the overlap test and the clamp — i.e. take an `FVector2D FrameOrigin`
(or take the frame as an `FBox2D`) alongside the size. `DriveSetOfMarkRenderer` supplies it
per surface: `SWindow::GetPositionInScreen()` of the captured window for `editor_chrome`,
and the game viewport widget's desktop position for `game`/`web` (the same
`GetTickSpaceGeometry().GetAbsolutePosition()` the element walk uses, taken on the
viewport's `SViewport`/`SWindow`, not the viewport's `FIntPoint` size).

Do NOT "fix" this by moving element geometry back to a surface-local space: the elements'
desktop space is load-bearing for input injection (`FDriveInput`), and `drive.list_windows`
already publishes desktop window rects. The conversion belongs at the drawing boundary.

## Related

- `B-drive-geometry-window-space-injected-as-desktop` (IN-REVIEW, High) — established that
  element geometry is desktop space in UE 5.3-5.8 and named the space in-band
  (`geometry.space:"desktop"`). This ticket is the consumer that never got the memo; it was
  found during that fix and deliberately left out of it as an unrelated code path.
- `B-drive-observe-collapsed-ancestor-reads-visible` (IN-REVIEW, High) — its fix zeroes a
  non-drawn element's rect, so such elements now land in `marks_omitted` via the
  `bOverlapsFrame`/too-small tests instead of being marked at stale coordinates. That is a
  strict improvement and independent of this defect.
- `B-drive-setofmark-writes-jpeg` — same renderer, unrelated (encode format).

## Workaround

Do not read positions off the Set-of-Mark image. Use `drive.observe`'s `elements[]` and
target by `handle`; treat the numbered screenshot as an inventory, not as a map. Treat
`marks_omitted` as "not proven offscreen".

## History
- `#1-marks-drawn-at-desktop-coords-on-local-bitmap` `OPEN` reporter — Filed out of the `B-drive-geometry-window-space-injected-as-desktop` fix on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `d5dfb11b`. Read verbatim in source, not reproduced live. `FDriveSetOfMarkLayout::BuildLayout` accepts a frame SIZE and no ORIGIN (`DriveSetOfMarkLayout.cpp:16-18`) and compares `Element.AbsolutePosition` against `[0,0]..[Width,Height]` (`:33-34`), while `FDriveSetOfMarkRenderer` passes only the captured bitmap's dimensions (`DriveSetOfMarkRenderer.cpp:172-174`) for a bitmap that is viewport-local on `game`/`web` (`FViewport::ReadPixels` + `GetSizeXY()`, `:110-134`) or window-local on `editor_chrome` (`FDriveEditorChrome::CaptureWindow`, `:102-108`). Element geometry is desktop space (confirmed in the sibling ticket: `SWidget::GetCachedGeometry()` -> `GetTickSpaceGeometry()` -> `PersistentState.DesktopGeometry`, which `SWidget::Paint` stores with the owning window's `GetPositionInScreen()` appended, `SWidget.cpp:1494-1495`), so every mark is displaced by the captured surface's desktop origin. Severity High rather than Medium because, unlike the injection path in the sibling ticket, the GAME surface is wrong in the ORDINARY configuration: PIE hosted in the level-editor viewport puts the viewport's top-left inside the editor frame even with the editor maximized at (0,0), and that offset was measured at (+37,+254) in that ticket's `#1`. Both consequences are silent — a mark drawn over the wrong widget, and an on-screen element reported in `marks_omitted` as if offscreen — which is the rubric's silent-wrong-data class. No fix attempted; deliberately out of scope of the geometry ticket's diff, which changed no coordinate values.
- `#2-live-repro-and-missing-umg` `OPEN` reporter - **Reproduced live**, UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`, PIE in the level viewport (viewport 1036x755 at desktop origin about (617,295), DPI scale 0.54). `drive.observe {screenshot_mode: file}` on `W_MyDrones`: marks were drawn at the raw desktop x/y (for example mark 6 at 736,484), so they land about 617 px right and 295 px below their elements and most were reported `marks_omitted`. Separate defect filed for the second thing the same image showed: it contains no UMG at all (`B-drive-observe-screenshot-omits-umg`). Cheap.
- `#3-frame-origin-subtracted` `IN-REVIEW` developer - Fixed at the drawing boundary as the ticket prescribes; element geometry stays desktop space. `FDriveSetOfMarkLayout::BuildLayout` takes a required `const FVector2D& FrameOrigin` and shifts every rect by `-FrameOrigin` before the overlap test and the clamp (`DriveSetOfMarkLayout.h/.cpp`). `FDriveSetOfMarkRenderer::CaptureAnnotated` supplies it per surface: game/web = the game viewport widget's `GetCachedGeometry().GetAbsolutePosition()` (the same cached geometry the element walk reads, and the rect `TakeScreenshot`/`ReadPixels` start at); editor_chrome = `SWindow::GetPositionInScreen()` of the captured window, returned through a new `FVector2D& OutDesktopOrigin` out-param on `FDriveEditorChrome::CaptureWindow` (its only caller is the renderer). Baseline (unmodified code, Linux host, PIE in the level viewport, viewport at desktop ~(74,160), frame 1982x1243): `drive.observe` drew marks 2-8 about 74 px right / 160 px low of their buttons and put the on-screen power button (desktop x 1983) and the bottom row (desktop y 1280) in `marks_omitted`: `marks_drawn:[2,3,4,7,8]`, `marks_omitted:[1,5,6,9,10,11,12]`. After: `marks_drawn:[1,2,3,4,7,8,9,10,12]`, `marks_omitted:[5,6,11]` (the three collapsed, zero-rect buttons), each box on its button in `DriveObserve_20260924_122722_276_0001.png`. Tests: `PinWright.drive.setofmark.FrameOriginShiftsDesktopRects` (headless, the measured geometry: shifted box, far-edge element drawn, left-of-frame element omitted) and the live `PinWright.drive.observe.ScreenshotIncludesUmgAndMarksAtSurfaceLocalCoords` (element placed 100 px into the PIE viewport in desktop space must be outlined at frame-local x=100). The six existing `PinWright.drive.setofmark.*` layout tests pass with `FVector2D::ZeroVector`. Results (same build, live editor, `system.run_tests`): the new live test plus `PinWright.drive.setofmark.*` (7) and `PinWright.drive.somrender.*` (4) = 12/12 Success, 0 `PINWRIGHT_ASSERTIONS_SKIPPED`; `PinWright.drive.editorchrome.*` 5/5 Success. Editor-chrome checked live too: `drive.observe {surface:editor_chrome}` on the editor window at desktop (70,27) boxed "Save Current Level" / "Browse To Level" exactly on those toolbar buttons (`DriveObserve_20260924_125747_757_0002.png`). Plugin commit `b75217d7`.
