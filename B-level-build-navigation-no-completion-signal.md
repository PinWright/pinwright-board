---
id: B-level-build-navigation-no-completion-signal
title: "level.build_navigation returned {started:true} with no completion signal"
status: DONE
severity: High
category: bug
tags: [async, jobs, level, navigation, no-completion-signal]
---

# level.build_navigation returned {started:true} with no completion signal

`level.build_navigation` triggered a navigation mesh rebuild and returned `{started:true}`. NavMesh builds can take tens of seconds on large levels; callers had no progress signal.

**Fix:** Handler now calls `Ctx.StartJob()` and registers a game-thread ticker that polls `FNavigationSystem::IsNavigationBuildInProgress()` once per tick. When the flag clears, the ticker removes itself and calls `FJobRegistry::CompleteJob(JobId, true, "Navigation build complete")`.

**Files:** `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:1545`.

## History
- `#1-no-completion-signal` `OPEN` reporter — NavMesh build kicked off, returned immediately, no completion event.
- `#2-poll-is-nav-build-in-progress` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Ticker polls `FNavigationSystem::IsNavigationBuildInProgress()` at 0.25 s interval.
- `#3-verified-completion-payload` `DONE` tester — Verified end-to-end: kicked `level.build_navigation` → ticket `j_20260427T023708_e77860ab`. `system.job_status` 6s later returned `status:"completed"` with `result:{nav_built:true, nothing_to_build:true}` (current map L_Core has no nav-relevant geometry to rebuild). Ticker poll path confirmed working; nav delegates fire `CompleteJob` correctly.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `BuildNavigationHandler.cpp` never existed; the verb is `LevelHandler.cpp:1545`, job at `:1558`, completion via `LevelBuildBinds.h:179`/`:190`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
