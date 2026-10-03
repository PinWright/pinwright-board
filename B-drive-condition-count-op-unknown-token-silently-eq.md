---
id: B-drive-condition-count-op-unknown-token-silently-eq
title: "drive.wait_for / drive.expect `count` condition: an unrecognized `count_op` token is silently treated as `eq`"
status: OPEN
severity: Medium
category: bug
tags: [drive, drive.wait_for, drive.expect, condition, count_op, argument-validation]
encounters: 1
lastSeen: 2026-10-02T00:00:00Z
---

# An unknown `count_op` silently becomes `eq`

## Symptom

`FDriveJson::ParseCondition` (`Source/PinWright/Private/Handlers/Drive/DriveJson.cpp`) calls
`CompareOpFromString(OpToken, OutCondition.CountOp)` and ignores its `false` return, so a `count`
condition with `count_op:"greater_than"` (or `">="`, `"ge"`) keeps the default `eq`. A caller
asking "wait until there are at least 3 rows" then waits for exactly 3, and reports `timeout` or a
wrong `met`. Found by source reading while fixing
`E-drive-condition-missing-target-silent-timeout`; not reproduced live.

## Ask

Refuse an unrecognized `count_op` with `CONDITION_INVALID` (naming the token and the accepted
`eq, ne, lt, lte, gt, gte`) through ParseCondition's `OutError`, before polling.

## History

- `#1-found-in-source` `OPEN` developer - Spotted in `ParseCondition` while closing the condition key
  set (plugin working tree on `10212ee4`).
