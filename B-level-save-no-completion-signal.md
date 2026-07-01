---
id: B-level-save-no-completion-signal
title: "level.save returned {started:true} with no completion signal"
status: DONE
severity: Medium
category: bug
tags: [async, jobs, level, save, no-completion-signal]
---

# level.save returned {started:true} with no completion signal

`level.save` triggered a level save and returned `{started:true}`. Save operations involve disk I/O and may trigger source control hooks; callers had no confirmation that the save had completed before issuing further edits.

**Fix:** Handler now calls `Ctx.StartJob()` and wraps the save call in an `AsyncTask(ENamedThreads::GameThread, ...)` that calls `FEditorFileUtils::SaveLevel`, then calls `FJobRegistry::CompleteJob` with `{saved: true/false}` from the result.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/Level/SaveLevelHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Level save returned immediately. Callers could not reliably sequence a save followed by a source control submit.
- `#2-asynctask-wrap` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. `AsyncTask` wraps the synchronous save call; on return `CompleteJob` is fired with the save result.
- `#3-verified-saved-payload` `DONE` tester — Verified end-to-end: kicked `level.save` → ticket `j_20260427T023602_1eb68a46` with `packageName:"/Game/System/FrontEnd/Maps/L_Core"`. `system.job_status` ~1.2 s later returned `status:"completed"` with `result:{saved:true}`. jobs.jsonl recorded `started` + `completed` events with matching payload.
