---
id: E-drive-wait-visible-before-clickable
title: "wait_for widget_visible is met while an opening menu is still animating in, and a drive.click fired at that moment is silently dropped (outcome wait_for_met / timeout, no error)"
status: OPEN
severity: Medium
category: ergonomic
tags: [drive, drive.click, drive.key, wait_for, widget_visible, animation, commonui, dropped-input]
encounters: 1
lastSeen: 2026-09-30T13:32:00Z
---

# `widget_visible` does not mean "accepts clicks yet"

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `adb239fd`, PIE (~10-12 fps). The PDS
in-game menu `W_HUD_DroneGameMenu` plays an `OnActivated` open animation.

- `drive.key {key:"Escape", wait_for:{type:"widget_visible", target:".../W_HUD_DroneGameMenu_C_2/W_PauseMenu/HomeButton/SCommonButton"}}`
  returned `wait_for_met` after 105-110 ms, 1 tick.
- The next call, `drive.click {handle: that HomeButton, os_input:true, wait_for:{widget_visible: the confirmation's Yes button}}`,
  ran to a 4000 ms `timeout`. The confirmation never appeared: the click was dropped. A second identical
  click about 10 s later opened the confirmation at once.
- The same pattern dropped a `RatesButton` click right after the menu became "visible".

`widget_visible` reads effective visibility, which is true from the first frame of the open animation,
before the button hit-tests or the activatable is ready for input. A caller that gates a click on it
gets a silent no-op: no error, only a timed-out follow-up condition.

**Expected:** a condition that means "ready for input" (for example `widget_interactable`, true once the
element is hit-test visible, enabled and not under a running open animation or an inactive activatable),
or a note on `widget_visible` in the drive wiki that it is not an input-readiness signal. A drive.click
whose target did not receive the press (no button pressed state, nothing changed) could also report that.

## History
- `#1-filed-pause-menu-animation` `OPEN` reporter - Filed during the PDS card #830 verification in PIE: HOME and Rates clicks gated on `widget_visible` were dropped while the in-game menu was still animating in. Cost: two retries and a log/ini check to find that the travel never started.
- `#2-stale-sweep-bump-medium` `OPEN` developer — Severity Low -> Medium. Still reproduces in source at plugin HEAD `10212ee4`: `widget_visible` reads only effective visibility (`docs/wiki-src/drive.md:47`), there is no input-readiness condition (no hit-test / activatable / open-animation check in `DriveConditionEval.cpp`), and the action verb's response carries no signal that the press reached nothing. Rubric: this is an agent trap that wastes real time on a common path, and the docs actively steer callers into it (`drive.md:47` presents `widget_enabled`/`widget_visible` as gates for "wait until the Create button becomes clickable"); every CommonUI activatable with an open animation hits it, and the dropped click surfaces only as a later timeout.
