---
id: B-editor-save-all-no-completion-signal
title: "editor.save_all returned {started:true} with no completion signal"
status: DONE
severity: Medium
category: bug
tags: [async, jobs, editor, save-all, no-completion-signal]
---

# editor.save_all returned {started:true} with no completion signal

`editor.save_all` triggered a bulk save of all dirty assets and returned `{started:true}`. Saving dozens of assets involves multiple disk writes and may trigger source-control hooks; callers could not sequence further operations safely.

**Fix:** Handler now calls `Ctx.StartJob()` and wraps `FEditorFileUtils::SaveDirtyPackages` in an `AsyncTask`. On return, `FJobRegistry::CompleteJob` is called with `{savedCount, failedCount}`.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/Editor/SaveAllHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Bulk save returned immediately with no signal of how many assets saved or whether any failed.
- `#2-asynctask-wrap` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. `AsyncTask` wraps `FEditorFileUtils::SaveDirtyPackages`; `CompleteJob` fires with `{savedCount, failedCount}`.
- `#3-verified-savedcount-payload` `DONE` tester — Verified: kicked `editor.save_all` → ticket `j_20260427T024453_8c8bce49`. `system.job_status` returned `status:"completed"` with `result:{success:true, savedCount:0, totalDirty:0}` (clean editor state). jobs.jsonl recorded `event:"started"` and `event:"completed"` with the same result payload.
