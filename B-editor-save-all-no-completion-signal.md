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

**Files:** `Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp:447`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Bulk save returned immediately with no signal of how many assets saved or whether any failed.
- `#2-asynctask-wrap` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. `AsyncTask` wraps `FEditorFileUtils::SaveDirtyPackages`; `CompleteJob` fires with `{savedCount, failedCount}`.
- `#3-verified-savedcount-payload` `DONE` tester — Verified: kicked `editor.save_all` → ticket `j_20260427T024453_8c8bce49`. `system.job_status` returned `status:"completed"` with `result:{success:true, savedCount:0, totalDirty:0}` (clean editor state). jobs.jsonl recorded `event:"started"` and `event:"completed"` with the same result payload.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. **Not just a path — this DONE ticket's fix has since been undone and nothing in it says so.** `SaveAllHandler.cpp` never existed; `editor.save_all` lives at `EditorCommandHandler.cpp:447` and is now fully **synchronous**: `check(IsInGameThread())` at `:452`, `Ctx.SendSuccess(Result)` at `:461`, no job envelope anywhere, and the registered summary reads “Runs synchronously… returns the result inline”. The body's `Ctx.StartJob()` / `AsyncTask` / `CompleteJob{savedCount, failedCount}` describes code that is gone, and `#3`'s tester evidence describes behaviour the binary no longer has. The change shipped under `E-save-all-sync-fast-path`; whether this ticket stays `DONE` wants a decision. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
