---
id: B-pipeline-run-ubt-no-completion-signal
title: "pipeline.run_ubt returned {started:true} with no completion signal"
status: DONE
severity: High
category: bug
tags: [async, jobs, pipeline, ubt, no-completion-signal]
---

# pipeline.run_ubt returned {started:true} with no completion signal

`pipeline.run_ubt` (pipeline domain alias for the UBT invocation) spawned a UBT process and returned `{started:true}` with no completion signal — identical to `system.run_ubt`.

**Fix:** Handler now calls `Ctx.StartJob()` and registers a game-thread ticker that polls the spawned process handle via `FPlatformProcess::GetProcReturnCode` until the process exits, then calls `FJobRegistry::CompleteJob`.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/Pipeline/RunUbtHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Pipeline domain UBT alias had the same missing-completion-signal bug as `system.run_ubt`.
- `#2-proc-poll-via-ticker` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Same proc-poll ticker pattern as `system.run_ubt`.
- `#3-skip-launches-ubt` `SKIP` tester — Live test would spawn a real UBT process (heavy, side effects on Intermediate/, can't safely cancel mid-link). Schema verified registered (`{target?, platform?, configuration?, extraArgs?}` discovery). Kickoff path inferred to share infrastructure with the other 14 verified handlers in this session (canonical ticket_id+monitor_path response). End-to-end UBT process completion not run.
- `#4-accepted-without-recheck` `DONE` tester — Accepted by user decision without further live verification; prior SKIP entry documents why the UBT process test was not re-run.
