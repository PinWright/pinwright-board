---
id: B-drive-observe-screenshot-omits-umg
title: "drive.observe's game-surface screenshot is a bare scene render: no UMG, no post-process blur, so the image contradicts the element list it annotates"
status: OPEN
severity: High
category: bug
tags: [drive, drive.observe, set-of-mark, screenshot, umg, pie, silent-wrong-data]
encounters: 2
lastSeen: 2026-09-24T03:10:00Z
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
