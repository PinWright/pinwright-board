---
id: E-drive-condition-missing-target-silent-timeout
title: "drive.wait_for accepts a condition with no `target` (an unknown `handle` key instead) and polls the full timeout to `outcome:timeout` rather than rejecting the argument"
status: DONE
severity: Medium
category: ergonomic
tags: [drive, drive.wait_for, drive.expect, condition, argument-validation, unknown-params, timeout]
encounters: 1
lastSeen: 2026-09-24T09:07:00Z
---

# A condition without `target` waits out the timeout

## Symptom

`drive.wait_for {surface:"game", instance_name:"W_OverallUILayout", condition:{type:"widget_visible",
handle:"W_OverallUILayout_C_0/.../NextButton_Step_8/SCommonButton"}, timeout_ms:15000}` returned
`{met:false, outcome:"timeout", elapsed_ms:15019}`. The widget became visible within that window (the
next `drive.observe` listed it `visible:true` and `drive.click` on it worked). The condition used
`handle`, the key every other drive verb takes, instead of `target`; the unknown key was ignored and the
missing required `target` was not reported, so the call burned 15 s and read as "never became visible".

## Ask

Reject a `widget_*` / `text_*` condition without `target` (and unknown condition keys) with
`INVALID_ARGUMENT` before polling, or accept `handle` as an alias of `target` since action verbs use it.

## History

- `#1-handle-key-ignored` `OPEN` reporter - Filed from a school lesson-attempt PIE session (Linux, host
  `/sdb-disk/src/unreal/unreal-fpv`, plugin `ba115afb`).
- `#2-re-rated` `OPEN` triage — Severity Low -> Medium. A condition missing `target` is not rejected, so the call reports `met:false, outcome:timeout` for a widget that was visible: silent wrong result (High class) triggered by an argument-shape slip rather than the normal path, so one level down to Medium.
- `#3-closed-condition-keys` `IN-REVIEW` developer — The missing-`target` half was already fixed by
  `ec169d19` (after the reporter's `ba115afb`): `{type:"widget_visible", handle:"..."}` is refused
  `CONDITION_INVALID`, but with a message that never named `handle`, and `handle` (or `expected`
  for `expected_text`, etc.) beside a valid `target` was still dropped silently. Now
  `FDriveJson::ParseCondition` (`Handlers/Drive/DriveJson.cpp/.h`) refuses any key outside
  `type, target, expected_text, expected_count, count_op, expected_bounds, severity`
  (case-insensitive, like FJsonObject lookup; via the shared `RejectUnknownKeys`) before polling, with a hint to use `target` when the key is `handle`, and
  returns the failing rule through a new `FString* OutError` (also threaded through
  `ParseSettleConfig`). Every caller sends that reason in its `CONDITION_INVALID` message:
  `DriveWaitHandler.cpp`, `DriveExpectHandler.cpp`, `DriveActionCommon.cpp` (action `wait_for`),
  `DriveWebHandlers.cpp` (web re-parse + web action `wait_for`). Chose the parser over the
  dispatcher nested-key gate: one choke point covers `drive.wait_for`, `drive.expect` and every
  action verb's `wait_for` (the gate would need an adoption entry per verb in
  `TestNestedParamKeyGate.cpp` and still could not enforce `target`). Param descriptions,
  `docs/wiki-src/drive.md` (Verification) and CHANGELOG (behaviour change) updated. Tests:
  `PinWright.drive.contract.ConditionUnknownKeyRefused` (parser + settle config),
  `PinWright.drive.actions.WaitForUnknownConditionKeyInvalid` (handler-level, the reported call
  and handle-beside-target on both verbs; fails on revert: the message lacks `'handle'`, and the
  handle+target shape reaches the live-UI pre-check instead of `CONDITION_INVALID`). Not done:
  an unknown `count_op` token is still silently treated as `eq`.
- `#4-verified-linux` `DONE` tester — Fix commit 49e4dfd6 (missing-target half from ec169d19, on master). Passed non-skipped in run3/full and run3/xdrive: `PinWright.drive.contract.ConditionUnknownKeyRefused` (the parser and the settle config refuse unknown condition keys) and `PinWright.drive.actions.WaitForUnknownConditionKeyInvalid` (the reported `{type:"widget_visible", handle:"..."}` call and the handle-beside-target shape, on drive.wait_for and drive.expect: `CONDITION_INVALID` before any polling, with a message naming `handle` and pointing at `target`). Acceptance: the ask's first option (reject a condition without `target`, and unknown condition keys, before polling) is met. No 15 s timeout now reads as "never became visible". Remaining, not in the ask: an unknown `count_op` token is still treated as `eq` (noted by the developer).
