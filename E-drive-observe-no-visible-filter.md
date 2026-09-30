---
id: E-drive-observe-no-visible-filter
title: "drive.observe interactables_only returns hidden (visible:false, stale geometry) elements ahead of the visible ones, and there is no visible-only filter, so max_bytes/max_elements cut the list before any on-screen control"
status: OPEN
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
