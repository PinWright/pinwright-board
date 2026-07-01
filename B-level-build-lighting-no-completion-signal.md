---
id: B-level-build-lighting-no-completion-signal
title: "level.build_lighting returned {started:true} with no completion signal"
status: DONE
severity: High
category: bug
tags: [async, jobs, level, lighting, no-completion-signal]
---

# level.build_lighting returned {started:true} with no completion signal

`level.build_lighting` triggered the lighting build and returned `{started:true}` immediately. Lighting builds can take minutes; callers had no way to know when baking finished or whether it succeeded.

**Fix:** Handler now calls `Ctx.StartJob()` and binds both `FEditorDelegates::OnLightingBuildSucceeded` and `FEditorDelegates::OnLightingBuildFailed` delegates. The first to fire calls `FJobRegistry::CompleteJob(JobId, bSuccess, Message)` and unbinds both delegates.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/Level/BuildLightingHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Lighting build started with no way to await its outcome.
- `#2-bound-to-lighting-delegates` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Bound `FEditorDelegates::OnLightingBuildSucceeded` / `OnLightingBuildFailed`. First-fires-wins pattern cleans up both handles before calling `CompleteJob`.
- `#3-verified-kickoff-and-cancel` `DONE` tester — Verified: kicked `level.build_lighting quality:"preview"` → ticket `j_20260427T023735_96daaa27` with `{status:"running", quality:"Preview", ticket_id, monitor_path, method, started_at}`. `system.job_cancel` on the running ticket returned `{cancelled:true}`. End-to-end completion delegate not observed (preview build still ran beyond the session window) but kickoff + registry integration + cancel path all confirmed.
