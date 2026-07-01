---
id: B-lighting-build-lighting-no-completion-signal
title: "lighting.build_lighting returned {started:true} with no completion signal"
status: DONE
severity: High
category: bug
tags: [async, jobs, lighting, no-completion-signal]
---

# lighting.build_lighting returned {started:true} with no completion signal

`lighting.build_lighting` (the `lighting` domain alias for the same operation as `level.build_lighting`) triggered a lighting build and returned `{started:true}` immediately with no way to await completion.

**Fix:** Handler now calls `Ctx.StartJob()` and binds `FEditorDelegates::OnLightingBuildSucceeded` / `FEditorDelegates::OnLightingBuildFailed`, same pattern as `level.build_lighting` and `level.build_level_lighting`.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/Lighting/BuildLightingHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Lighting domain alias had the same missing-completion-signal bug as the level domain counterpart.
- `#2-bound-to-lighting-delegates` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Bound `FEditorDelegates::OnLightingBuildSucceeded` / `OnLightingBuildFailed`.
- `#3-verified-alias-kickoff` `DONE` tester — Verified: invoked the alias via raw RPC `lighting.build_lighting quality:"preview"` → ticket `j_20260427T024114_71c4176d`, canonical kickoff JSON, `system.job_cancel` returned `{cancelled:true}`. Same registry-integrated path as `level.build_lighting` which was also passed in this session.
