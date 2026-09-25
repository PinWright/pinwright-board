---
id: B-drive-observe-screenshot-omits-umg
title: "drive.observe's game-surface screenshot is a bare scene render: no UMG, no post-process blur, so the image contradicts the element list it annotates"
status: IN-REVIEW
severity: High
category: bug
tags: [drive, drive.observe, set-of-mark, screenshot, umg, pie, silent-wrong-data]
encounters: 3
lastSeen: 2026-09-24T09:00:00Z
---

# The Set-of-Mark image shows a different frame from the one on screen

UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`, PIE hosted in the level-editor viewport on `/Game/System/FrontEnd/Maps/L_Core`.

`drive.observe {instance_name: W_OverallUILayout, screenshot_mode: file}` returned element lists that
matched the live UI (the login overlay in one call, the `W_MyDrones` grid in another), but both PNGs
(`Saved/Screenshots/Drive/DriveObserve_20260923_224522_589_0001.png` and `..._225751_466_0003.png`)
show only the stadium scene: no widgets, and not even the menu's background blur. An
`editor.screenshot` taken at the same moment (`captureMode: nativeBackBuffer`) shows the overlay and
the grid correctly. The first call also happened while `W_ForcedLogoutPopup` was on screen; the
popup is absent from the image too.

Impact: the image is the part of the observation an agent looks at first. Reading it, an agent concludes the
menu is not there, while the element list says it is. Combined with
`B-set-of-mark-layout-ignores-surface-origin`, the marks also land on empty scene pixels.

**Workaround:** `screenshot: false` on `drive.observe`, plus `editor.screenshot` for pixels.
**Fix:** capture the composited back buffer (the path `editor.screenshot` uses for `nativeBackBuffer`)
for the game surface, or render the game-layer Slate tree over the scene as the exact-size path does.

## History
- `#1-som-image-has-no-umg` `OPEN` reporter - Seen on two observes in a login regression pass; UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Cheap (switched to editor.screenshot), but the misleading image is on the main observe path.
- `#2-school-login-pass-same-symptom` `OPEN` reporter - Seen again, UE 5.8, host `unreal-fpv-new`, plugin `8748c637`: first `drive.observe {instance_name:"W_OverallUILayout", screenshot_mode:"file"}` of a PIE session on `L_Core` returned `DriveObserve_20260924_050923_813_0001.png` showing only the stadium scene while the element list (and an `editor.screenshot` taken seconds later) had the full `W_LoginOverlay` name form on screen. Cheap: switched to `editor.screenshot {width:1920,height:1080}` for every capture.
- `#3-linux-school-attempt-pass` `OPEN` reporter - Seen again on Linux, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `ba115afb`: three `drive.observe {surface:"game", instance_name:"W_OverallUILayout", screenshot_mode:"file"}` calls (school name login form, then a lesson briefing with its «ОК» button) all wrote scene-only PNGs with empty `marks_drawn` (every mark in `marks_omitted`), while `drive.click` on the listed handles actuated the real buttons. Also present on plugin `ba115afb`, so not fixed by the related `B-screenshot-omits-umg-overlay` work.
- `#4-back-buffer-capture` `IN-REVIEW` developer - Root cause: `FDriveSetOfMarkRenderer::CaptureAnnotated` read the game/web surface with `FViewport::ReadPixels`, the scene render target only; the `B-screenshot-omits-umg-overlay` fix changed the shared `CaptureGameViewportToPngFile` path and explicitly left this renderer's inline readback alone. Fix (`Source/PinWright/Private/Handlers/Drive/DriveSetOfMarkRenderer.cpp`): the game/web capture now calls `PinWrightScreenshotUtils::TakeSlateScreenshot` on `GameViewport->GetGameViewportWidget()`, the same composited back-buffer read `editor.screenshot` reports as `captureMode: nativeBackBuffer`, and keeps `ReadPixels` only as the fallback when no back buffer can be read (headless / `-RenderOffScreen`), mirroring `CaptureGameViewportToPngFile`. Wiki: `docs/wiki-src/drive.md` states what the image is on each surface. Baseline (unmodified code, Linux host, PIE in the level viewport on `L_Core`): `drive.observe {surface:game, instance_name:W_OverallUILayout, screenshot_mode:file}` wrote `DriveObserve_20260924_075342_820_0001.png` showing only the stadium scene while `editor.screenshot` seconds later (`nativeBackBuffer`) showed the full front-end menu. After: the same call writes `DriveObserve_20260924_122722_276_0001.png` with the menu, logo, version text and buttons present and each mark on its button. Test: `PinWright.drive.observe.ScreenshotIncludesUmgAndMarksAtSurfaceLocalCoords` (`Tests/Drive/TestDriveObserveScreenshotComposite.cpp`) owns a PIE session via latent commands, adds a full-viewport green overlay through `AddViewportWidgetContent`, and asserts the annotated capture reads green; the scene-only readback cannot. First run of it failed with 0/5 green because the host's loading screen (ZOrder 10000) was still up over a 9999 overlay, so the overlay ZOrder is 1000000. Not fixed here: the renderer still encodes via `ThumbnailCompressImageArray` (JPEG bytes), which is `B-drive-setofmark-writes-jpeg`. Results (same build, live editor, `system.run_tests`): the new live test plus `PinWright.drive.setofmark.*` (7) and `PinWright.drive.somrender.*` (4) = 12/12 Success, 0 `PINWRIGHT_ASSERTIONS_SKIPPED`; `PinWright.drive.editorchrome.*` 5/5 Success. Plugin commit `b75217d7`.
