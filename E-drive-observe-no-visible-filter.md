---
id: E-drive-observe-no-visible-filter
title: "drive.observe interactables_only returns hidden (visible:false, stale geometry) elements ahead of the visible ones, and there is no visible-only filter, so max_bytes/max_elements cut the list before any on-screen control"
status: DONE
severity: Medium
category: ergonomic
tags: [drive, drive.observe, interactables-only, visibility, element-list, spill, max-bytes]
encounters: 4
lastSeen: 2026-09-30T12:26:00Z
---

# `interactables_only` keeps every collapsed widget, and nothing filters them out

`drive.observe {instance_name:"W_OverallUILayout", interactables_only:true}` on the PDS main menu returns
every interactable widget in the tree, including whole collapsed screens (`W_LoginOverlay`'s `W_CreateUser`,
`W_Login`, `W_RestorePassword`, ...), each with `visible:false`, `geometry.stale:true` and a 0x0 rect.
They come first in tree order, so on a login overlay the six visible name-form controls sit behind ~40
hidden ones.

- With `max_bytes: 12000` the response kept only hidden elements (`omitted_count: 48`) and dropped every
  visible control the caller wanted.
- With `max_bytes: 0` the payload is 22-100 KB and spills to a file on every observe; each needed a local
  script to print the `visible:true` rows.

Repro: UE 5.8, host `unreal-fpv-new`, plugin `8748c637`, PIE on `/Game/System/FrontEnd/Maps/L_Core` with the
login overlay open; `drive.observe {instance_name:"W_OverallUILayout", interactables_only:true, max_bytes:12000}`.

**Workaround:** `max_bytes: 0`, then filter the spilled JSON for `"visible": true`.
**Fix:** a `visible_only` flag (or default hidden elements out of `interactables_only`, since a hidden
control is not actionable), applied before the `max_elements` / `max_bytes` caps.

## History
- `#1-hidden-elements-crowd-out-visible` `OPEN` reporter - Found in a school-computer login verification pass (host `unreal-fpv-new`, plugin `8748c637`): about 12 observes, each spilled or truncated to hidden-only rows; worked around with a local filter script. Cheap per call, recurring on every observe of a CommonUI menu that keeps inactive screens in the tree.
- `#2-truncated-before-visible-again` `OPEN` reporter - Second sighting (UE 5.8, PDS PIE, school-computer compatibility check). `drive.observe {instance_name:"W_OverallUILayout_C_0", interactables_only:true, max_elements:40}` returned 40 rows, all `visible:false` (hidden W_CreateUser / W_Login / W_FastUserCreateAndLogin fields); the on-screen W_SchoolNameLogin controls came only with `max_elements:400` (83 rows, 8 visible) and a local visible filter. Also tried an ad-hoc `filter` param first (UNKNOWN_PARAMS), which is the label filter `E-drive-observe-no-label-filter` asks for.
- `#3-hidden-only-on-login-and-briefing` `OPEN` reporter - Third sighting (UE 5.8, host `unreal-fpv`, plugin `61c243f5`, PIE cross-version check against an old backend). `drive.observe {instance_name:"W_OverallUILayout", interactables_only:true, max_bytes:6000}` on the password login overlay returned only hidden `W_CreateUser` rows (`omitted_count: 66`); the visible `W_Login` fields needed `max_bytes:0`, a spill file and a local visible filter. The Training02 briefing (`NextButton_Step_*`) needed the same filter to find which step button was on screen.
- `#4-re-rated` `OPEN` triage — Severity Low -> Medium. Impact class is Medium, not pure friction: under `max_bytes`/`max_elements` the response drops every visible control, so the only route is `max_bytes:0` plus a spill and a local filter script on each observe (about 12 observes in `#1`); `drive.observe` on CommonUI menus is a normal drive path, so no reach bump-down applies.
- `#5-pause-menu-spills-500k` `OPEN` reporter - Fourth sighting, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `adb239fd`, PIE freeflight with the in-game menu `W_HUD_DroneGameMenu` open. `drive.observe {instance_name:"W_OverallUILayout", interactables_only:true}` returned 106-137k characters (spilled to file) with only 9 visible elements; with `interactables_only:false` on the Rates page it was 496k characters (676k on disk), because every collapsed sibling page (`W_ControllerAxesPanel`, `W_MeteoSettingScreen`, `W_PauseMenuSettingScreen`, `W_DroneSelect_EditDrone`) is listed. Every observe in the session needed a Python pass over the spill file filtering `visible:true` to find the 20-odd controls on screen.
- `#6-visible-only-param` `IN-REVIEW` developer - Added opt-in `visible_only` (boolean, default false) to `drive.observe` on every surface (game, editor_chrome, web). `FDriveHandlerCommon::FilterObservationElements` drops elements whose effective `bVisible` is false (collapsed/hidden ancestor or wholly clipped, per the clip-aware DriveElementFactory) right after the `interactables_only` filter and BEFORE the `max_elements`/`max_bytes` caps; the drop is a filter, not counted in `omitted_count`. Chose a separate flag over changing `interactables_only`'s default because `visible:false` also covers controls clipped out of a scroll view, which `interactables_only` callers may still want listed; `interactables_only:true, visible_only:true` gives the on-screen actionable set. Files: `Source/PinWright/Private/Handlers/Drive/DriveHandlerCommon.{h,cpp}` (trailing defaulted `bVisibleOnly` on `FilterObservationElements` and `BuildObservation`), `DriveObserveHandler.cpp` (param decl + read), `DriveWebHandlers.cpp` (web observe path), new `Tests/Drive/TestDriveObserveVisibleOnly.cpp`, `docs/wiki-src/drive.md`, `CHANGELOG.md`. Tests: `PinWright.drive.observe.VisibleOnlyKeepsOnScreenUnderCaps` (pure filter: hidden-first list under `max_elements:2` / `max_bytes:1` keeps the visible rows; fails if the filter is removed or moved after the caps), `PinWright.drive.observe.VisibleOnlyParamDropsHiddenEditorChrome` (real dispatcher, so the param allowlist runs, over a fixture SWindow selected by `window_title` holding a visible leaf and a leaf under a Collapsed border; precondition asserts the unfiltered observe lists the collapsed leaf as `visible:false`, then `visible_only` must be accepted, non-empty, all visible, keep the visible leaf and drop the collapsed one).
- `#7-verified-linux` `DONE` tester — Fix commit af1dd7b7. Passed non-skipped in run3/full and run3/xdrive: `PinWright.drive.observe.VisibleOnlyKeepsOnScreenUnderCaps` (a hidden-first list under `max_elements:2` and `max_bytes:1` keeps the visible rows; the filter runs before the caps) and `PinWright.drive.observe.VisibleOnlyParamDropsHiddenEditorChrome` (real dispatcher, so the param allowlist runs; `visible_only` is accepted and drops a leaf under a Collapsed border that the unfiltered observe lists as `visible:false`). Acceptance: the asked-for `visible_only` flag exists and is applied before `max_elements`/`max_bytes`, so a CommonUI menu's on-screen controls are no longer cut by hidden rows. Limits: the game (UMG) surface is covered through the shared FilterObservationElements, not a live PIE observe of W_OverallUILayout; the web observe path is untested.
