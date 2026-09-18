---
id: B-commit-failure-rollback-test-red-on-linux
title: "CommitFailureRollsBack fails on Linux: the deliberate move failure is not declared as an expected error"
status: OPEN
severity: Low
category: bug
tags: [tests, linux, asset-dump]
---

# `CommitFailureRollsBack` fails on Linux

`PinWright.utils.asset_dump_writer.CommitFailureRollsBack`
(`Source/PinWright/Private/Tests/Utility/TestAssetDumpWriter.cpp`) provokes a
commit failure on purpose: it puts a *file* where the transaction needs a
*directory*, then asserts the transaction rolls back. All of its own assertions
pass. The test is red anyway, because the engine's own failure path logs

```
LogFileManager: Error: Error moving file '<tmp>/blocked/child.txt' to '<dump>/blocked/child.txt'
```

and the automation framework promotes any `Error`-level log during a test into a
test failure. The test never calls `AddExpectedError` for it. On Linux
`IFileManager::Move` also retries for ~5 s first, so the test takes ~5 s and
prints five `Warning: MoveFile was unable to move` lines before the error.

**Evidence:** `Saved/PinWright/test-runs/dumproot_2026-09-18/automation.log` on
the reporting host — 17 tests, 16 pass, this one fails with the `LogFileManager`
error as its only reported event.

Not caused by the `ResolveDumpRoot` fix (`B-dump-root-double-prefixes-project-dir`),
but only *reachable* after it: before that fix the scratch root resolved to an
unwritable path on this host, so the move was never attempted and the test failed
on its rollback assertions instead.

**Why it is not simply "add `AddExpectedError`":** an expected message with
`Occurrences = 0` still fails the test when the message does *not* appear
(`AutomationTest.cpp:1840-1850`, "Expected suppressed ... did not occur"), so a
blanket declaration would turn this green on Linux and potentially red on a
platform or engine version whose `Move` fails without that log line. The fix
needs a check across the supported 5.3-5.8 matrix on both platforms, or a
platform-guarded declaration.

## History
- `#1-found-on-linux` `OPEN` reporter — Found while verifying the `ResolveDumpRoot` fix on a Linux host; the test group is outside the `PinWright.asset.dump` filter, which is why the wrap-up runs never showed it. All in-test assertions pass; the failure is entirely the undeclared engine `LogFileManager: Error` from the deliberately blocked move.
