---
id: B-level-save-as-no-completion-signal
title: "level.save_as returned {started:true} with no completion signal"
status: DONE
severity: Medium
category: bug
tags: [async, jobs, level, save-as, no-completion-signal]
---

# level.save_as returned {started:true} with no completion signal

`level.save_as` triggered a save-as operation and returned `{started:true}`. Like `level.save`, callers had no confirmation of completion before proceeding to reference the newly-named asset.

**Fix:** Handler now calls `Ctx.StartJob()` and wraps `FEditorFileUtils::SaveLevelAs` in an `AsyncTask`. On completion, `FJobRegistry::CompleteJob` is called with `{newPath, saved}`.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/Level/SaveLevelAsHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Save-as returned immediately; callers couldn't reliably use the new asset path.
- `#2-asynctask-wrap` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. `AsyncTask` wraps save-as; `CompleteJob` fires with `{newPath, saved}`.
- `#3-skip-creates-asset` `SKIP` tester — Cannot test live without creating a new map asset (would mutate project state). Schema confirmed registered. Same `Ctx.StartJob()` + `AsyncTask` pattern as `level.save` which was verified end-to-end this session (`{saved:true}` payload, registry/jsonl events).
- `#4-accepted-without-recheck` `DONE` tester — Accepted by user decision without further live verification; prior SKIP entry documents why the map-asset creation test was not re-run.
