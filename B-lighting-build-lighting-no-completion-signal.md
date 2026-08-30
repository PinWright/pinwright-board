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

**Files:** `Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp:610`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Lighting domain alias had the same missing-completion-signal bug as the level domain counterpart.
- `#2-bound-to-lighting-delegates` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Bound `FEditorDelegates::OnLightingBuildSucceeded` / `OnLightingBuildFailed`.
- `#3-verified-alias-kickoff` `DONE` tester — Verified: invoked the alias via raw RPC `lighting.build_lighting quality:"preview"` → ticket `j_20260427T024114_71c4176d`, canonical kickoff JSON, `system.job_cancel` returned `{cancelled:true}`. Same registry-integrated path as `level.build_lighting` which was also passed in this session.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `Handlers/Lighting/` never existed as a directory; `lighting.build_lighting` is `Handlers/Environment/LightingHandler.cpp:610`, job at `:669`, completion at `:662`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
