---
id: B-drive-hover-no-umg-mouse-enter
title: "drive.hover moves the cursor over a PIE UUserWidget but its OnMouseEnter never fires (hover-driven dropdown stays closed)"
status: OPEN
severity: Medium
category: bug
tags: [drive, drive.hover, pie, pie-in-viewport, umg, on-mouse-enter, slate-injection, hover]
encounters: 1
lastSeen: 2026-09-23T18:30:00Z
---

# drive.hover does not trigger UMG OnMouseEnter in PIE

PIE hosted in the level-editor viewport, `/Game/System/FrontEnd/Maps/L_Core`. `W_LobbyLoginButton` (a UUserWidget inside W_LyraFrontEnd) plays its `OnHovered` animation from `Event OnMouseEnter`. `drive.hover` on its `NickName` text (also on `tb_ProfileSub`, and after first hovering another button so there is a real enter transition) returns `no_change_within_budget`; `drive.observe` shows the dropdown buttons still translated off-screen; `drive.input_state` reports `slate_active:false` with the OS cursor at the element center. `drive.click` on the same surface works.

**Workaround:** a PIE-only Python poke calling `play_animation_forward(OnHovered)` on the live widget.

**Source (8748c637):** `drive.hover` (`DriveActionHandlers.cpp:172-202`) calls `FDriveInput::HoverAt` = `MoveTo` (`DriveInput.h:84`, `DriveInput.cpp:244-257`), which sets the OS cursor and sends one `ProcessMouseMoveEvent` (`DriveInput.cpp:117-140`). `os_input` is Linux/X11-only, so there is no alternative injection path on Windows.

**Related:** `B-drive-click-misses-pie-game-viewport` (OPEN) `#3` saw `drive.hover` inert alongside `drive.click` with PIE hosted in the level viewport; here clicks worked, so the mouse-capture explanation there does not cover this case. `B-drive-geometry-window-space-injected-as-desktop` (window-space vs desktop coordinates) is ruled out here because clicks at the same centers landed.

## History
- `#1-hover-no-enter` `OPEN` reporter — UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Possibly the same inactive-application path as `B-drive-click-misses-pie-game-viewport`; filed separately because clicks worked here.
