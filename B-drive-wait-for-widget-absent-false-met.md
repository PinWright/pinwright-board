---
id: B-drive-wait-for-widget-absent-false-met
title: "drive.click wait_for {type: widget_absent, target: <UMG widget name>} reports wait_for_met on tick 1 while the widget is still on screen"
status: IN-REVIEW
severity: High
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
- `#2-re-rated` `OPEN` triage — Severity Medium -> High. Silent false success on a normal path: `widget_absent` is how an agent confirms a dialog closed, and source confirms (`DriveConditionEval.cpp` `WidgetAbsent`: `bMet = (Match == nullptr)`, match is exact full handle or label only) that any target naming a widget by its UMG name reports `wait_for_met` while the widget is on screen; no reach modifier.
- `#3-segment-match-and-typed-refusal` `IN-REVIEW` developer - Fixed at the matcher. `FDriveConditionEval::ElementMatchesTarget` (`Source/PinWright/Private/Handlers/Drive/DriveConditionEval.cpp`) now also matches a target that is a whole `/`-delimited run of the handle's segments (case-sensitive). That covers a UMG widget name (`OkButton` in `W_Popup_C_0/OkButton/SCommonButton`), a sub-path, or the trailing Slate type. So `widget_absent OkButton` reads present while the button is on screen, in `drive.expect`, `drive.wait_for` and action `wait_for` alike. `FindMatch` prefers an exact handle or label match over a sub-path match. Unresolvable specs are now typed errors. `FDriveJson::ParseCondition` rejects an empty `target` for every type except `journal_severity`, and callers get `CONDITION_INVALID` with a message naming the missing target. `FDriveActionCommon::RunAction` refuses an action whose `wait_for` is `widget_absent` with a target that matches no element in the pre-action baseline (`IsAbsenceUnverifiable`). The error is `CONDITION_INVALID` and nothing is injected. The reporter's literal proposal (`met:false` whenever the target matches nothing) was not taken, because it would make `widget_absent` impossible to satisfy. Standalone `drive.wait_for` and `drive.expect` keep the rule that absent means met: a dialog that already closed is a legitimate answer there, and they have no pre-action baseline. Tests: `PinWright.drive.condition.HandleSegmentMatch`, `PinWright.drive.condition.AbsenceNeedsBaselineMatch`, `PinWright.drive.contract.ConditionTargetRequired`. Docs: `docs/wiki-src/drive.md` Verification section.
