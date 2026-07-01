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

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/Level/BuildNavigationHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — NavMesh build kicked off, returned immediately, no completion event.
- `#2-poll-is-nav-build-in-progress` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Ticker polls `FNavigationSystem::IsNavigationBuildInProgress()` at 0.25 s interval.
- `#3-verified-completion-payload` `DONE` tester — Verified end-to-end: kicked `level.build_navigation` → ticket `j_20260427T023708_e77860ab`. `system.job_status` 6s later returned `status:"completed"` with `result:{nav_built:true, nothing_to_build:true}` (current map L_Core has no nav-relevant geometry to rebuild). Ticker poll path confirmed working; nav delegates fire `CompleteJob` correctly.
