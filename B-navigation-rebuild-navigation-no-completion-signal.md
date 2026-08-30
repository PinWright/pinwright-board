---
id: B-navigation-rebuild-navigation-no-completion-signal
title: "navigation.rebuild_navigation returned {started:true} with no completion signal"
status: DONE
severity: High
category: bug
tags: [async, jobs, navigation, no-completion-signal]
---

# navigation.rebuild_navigation returned {started:true} with no completion signal

`navigation.rebuild_navigation` (navigation domain alias) triggered a NavMesh rebuild and returned `{started:true}` with no completion signal, identical to the `level.build_navigation` issue.

**Fix:** Handler now calls `Ctx.StartJob()` and registers a game-thread ticker that polls `FNavigationSystem::IsNavigationBuildInProgress()` until the flag clears, then calls `FJobRegistry::CompleteJob`.

**Files:** `Source/PinWright/Private/Handlers/AI/NavigationHandler.cpp:300`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Navigation domain alias had the same missing-completion-signal bug.
- `#2-poll-is-nav-build-in-progress` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Same `IsNavigationBuildInProgress` poll pattern as `level.build_navigation`.
- `#3-verified-alias-completes` `DONE` tester — Verified end-to-end: invoked alias via raw RPC `navigation.rebuild_navigation` → ticket `j_20260427T024045_6051ca8f`. `system.job_status` 6s later returned `status:"completed"` with `result:{nav_built:true, nothing_to_build:true}` (same payload as `level.build_navigation`). Alias and the level-domain handler share the same ticker poll path.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `Handlers/Navigation/` never existed; the verb is `Handlers/AI/NavigationHandler.cpp:300`, job at `:337`, completion at `:331`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
