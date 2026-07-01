---
id: B-system-run-ubt-no-completion-signal
title: "system.run_ubt returned {started:true} with no completion signal"
status: DONE
severity: High
category: bug
tags: [async, jobs, system, ubt, no-completion-signal]
---

# system.run_ubt returned {started:true} with no completion signal

`system.run_ubt` spawned the Unreal Build Tool process and returned `{started:true}` immediately. The spawned process ran to completion (or crashed) with no signal back to the MCP caller, forcing agents to either wait an arbitrary time or check log files externally.

**Fix:** Handler now calls `Ctx.StartJob()` and registers a game-thread ticker that polls the spawned process handle (`FPlatformProcess::GetProcReturnCode`) once per tick. When the process exits the ticker removes itself and calls `FJobRegistry::CompleteJob(JobId, ReturnCode == 0, ...)` with the exit code and stdout tail in the payload.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/System/RunUbtHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — UBT process spawned, handler returned immediately. No way to know when build finished or whether it succeeded.
- `#2-proc-poll-via-ticker` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Ticker polls `FPlatformProcess::GetProcReturnCode` every 0.5 s. On process exit, calls `CompleteJob` with `{exitCode, success}`.
- `#3-skip-launches-ubt` `SKIP` tester — Same SKIP rationale as `B-pipeline-run-ubt-no-completion-signal` (sibling alias): live invocation spawns a real UBT process which is heavy, mutates Intermediate/, and is unsafe to cancel mid-link. Schema and registration confirmed via discovery. End-to-end test deferred.
- `#4-accepted-without-recheck` `DONE` tester — Accepted by user decision without further live verification; prior SKIP entry documents why the UBT process test was not re-run.
