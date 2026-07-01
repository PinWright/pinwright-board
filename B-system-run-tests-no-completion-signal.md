---
id: B-system-run-tests-no-completion-signal
title: "system.run_tests returned {started:true} with no completion signal"
status: DONE
severity: High
category: bug
tags: [async, jobs, system, automation, no-completion-signal]
---

# system.run_tests returned {started:true} with no completion signal

`system.run_tests` launched the automation test runner and returned `{started:true}` immediately, but never signalled when tests finished or how many passed/failed. Callers had no way to wait for results without polling unrelated log output.

**Fix:** Handler now calls `Ctx.StartJob()` to register a job and return `{jobId, started:true}`. The completion delegate is bound to `IAutomationControllerManager::OnTestsComplete`, which fires after the full test run finishes. On completion, `FJobRegistry::CompleteJob` is called with pass/fail counts from the controller.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/System/RunTestsHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Handler returned `{started:true}` with no completion event. Callers had to guess when tests were done.
- `#2-bound-to-ontestscomplete` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Bound `IAutomationControllerManager::OnTestsComplete` delegate to `FJobRegistry::CompleteJob(JobId, ...)`. Results include `{passed, failed, skipped}` counts in completion payload.
- `#3-skip-completion-not-observed` `SKIP` tester — Kickoff PASSED: invoked `system.run_tests filter:"DefinitelyNoTestMatchesThis_xyzzy"` → ticket `j_20260427T024506_0d3c9336` with canonical kickoff JSON including `command:"Automation RunTests …"`. After ~15 s `system.job_status` still reported `status:"running"`; OnTestsComplete didn't fire on a non-matching filter, so the completion path could not be observed within this session. `system.job_cancel` returned `{cancelled:true}`. Kickoff + cancel verified; completion delegate path not exercised live.
- `#4-accepted-without-recheck` `DONE` tester — Accepted by user decision without further live verification; prior SKIP entry documents kickoff/cancel evidence and why the completion path was not re-run.
