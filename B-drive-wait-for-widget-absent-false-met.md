---
id: B-drive-wait-for-widget-absent-false-met
title: "drive.click wait_for {type: widget_absent, target: <UMG widget name>} reports wait_for_met on tick 1 while the widget is still on screen"
status: OPEN
severity: Medium
category: bug
tags: [drive, drive.click, wait_for, widget_absent, condition, false-success, pie]
encounters: 1
lastSeen: 2026-09-23T20:55:00Z
---

# widget_absent is met by a widget that is present

UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`, PIE on `L_Core`. `W_ForcedLogoutPopup_C_0` was on screen with its button
`W_ForcedLogoutPopup_C_0/OkButton/SCommonButton` (label `Ок`) visible and interactable in `drive.observe`.

`drive.click {handle: W_ForcedLogoutPopup_C_0/OkButton/SCommonButton, instance_name: W_ForcedLogoutPopup,
wait_for: {type: widget_absent, target: OkButton}, timeout_ms: 3000}` returned `outcome: wait_for_met`,
`condition_met: true`, `elapsed_ms: 6`, `ticks: 1`. The click itself had not landed (an occluding
window, see `B-drive-click-misses-pie-game-viewport` `#4`). A screenshot right after still showed the
popup. So the condition matched nothing named `OkButton` and treated "no match" as "absent", although
the handle contains `OkButton` as a path segment.

Impact: `widget_absent` is how an agent confirms that a dialog closed. A condition that is met whenever the
target string fails to match is a silent false success.

**Fix:** match `target` against handle path segments (the UMG widget name) as well as label and type. When
`target` matches no element, return `met: false` with a detail saying so. Report `matched: 0` in the result.

## History
- `#1-absent-met-while-present` `OPEN` reporter - UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Cheap (caught by the next screenshot).
