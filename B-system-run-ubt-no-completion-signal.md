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

**Files:** `Source/PinWright/Private/Handlers/System/SystemControlHandler.cpp:366`.

## History
- `#1-no-completion-signal` `OPEN` reporter — UBT process spawned, handler returned immediately. No way to know when build finished or whether it succeeded.
- `#2-proc-poll-via-ticker` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Ticker polls `FPlatformProcess::GetProcReturnCode` every 0.5 s. On process exit, calls `CompleteJob` with `{exitCode, success}`.
- `#3-skip-launches-ubt` `SKIP` tester — Same SKIP rationale as `B-pipeline-run-ubt-no-completion-signal` (sibling alias): live invocation spawns a real UBT process which is heavy, mutates Intermediate/, and is unsafe to cancel mid-link. Schema and registration confirmed via discovery. End-to-end test deferred.
- `#4-accepted-without-recheck` `DONE` tester — Accepted by user decision without further live verification; prior SKIP entry documents why the UBT process test was not re-run.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. **Not just a path — the payload the body promises does not exist.** `RunUbtHandler.cpp` never existed; `system.run_ubt` is `SystemControlHandler.cpp:366`, job at `:446`. The body promises “the exit code and stdout tail”; `Handlers/BuildTools/ProcPollBind.h:37-38` sets exactly `{exit_code}`, and the plugin knows — `SystemControlHandler.cpp:448` comments “polls only the child's exit code (no output pipe)”. **The registration at `:366` still advertises “capture stdout/stderr”**, so the false promise is live in the wiki. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
