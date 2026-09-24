---
id: E-drive-condition-missing-target-silent-timeout
title: "drive.wait_for accepts a condition with no `target` (an unknown `handle` key instead) and polls the full timeout to `outcome:timeout` rather than rejecting the argument"
status: OPEN
severity: Low
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
