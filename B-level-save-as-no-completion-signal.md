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

**Files:** `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:388`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Save-as returned immediately; callers couldn't reliably use the new asset path.
- `#2-asynctask-wrap` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. `AsyncTask` wraps save-as; `CompleteJob` fires with `{newPath, saved}`.
- `#3-skip-creates-asset` `SKIP` tester — Cannot test live without creating a new map asset (would mutate project state). Schema confirmed registered. Same `Ctx.StartJob()` + `AsyncTask` pattern as `level.save` which was verified end-to-end this session (`{saved:true}` payload, registry/jsonl events).
- `#4-accepted-without-recheck` `DONE` tester — Accepted by user decision without further live verification; prior SKIP entry documents why the map-asset creation test was not re-run.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `SaveLevelAsHandler.cpp` never existed; the verb is `LevelHandler.cpp:388`, job at `:494`, completion at `:491`. Payload drift worth knowing: the body says `{newPath, saved}`; the real shape is `{saved, levelPath}` (`:483-489`) — `newPath` does not exist. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
