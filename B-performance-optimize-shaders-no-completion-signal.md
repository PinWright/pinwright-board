---
id: B-performance-optimize-shaders-no-completion-signal
title: "performance.optimize_shaders returned {started:true} with no completion signal"
status: DONE
severity: Medium
category: bug
tags: [async, jobs, performance, shaders, no-completion-signal]
---

# performance.optimize_shaders returned {started:true} with no completion signal

`performance.optimize_shaders` triggered shader compilation and returned `{started:true}`. Shader compilation can run for minutes; callers had no way to await completion.

**Fix:** Handler now calls `Ctx.StartJob()` and registers a game-thread ticker that polls `GShaderCompilingManager->IsCompiling()` once per second. When the flag clears the ticker calls `FJobRegistry::CompleteJob(JobId, true, "Shader compilation complete")`.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/Performance/OptimizeShadersHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Shader compile kicked off, returned immediately with no completion event.
- `#2-poll-gshadercompilingmanager` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Ticker polls `GShaderCompilingManager->IsCompiling()` at 1 s interval until false.
- `#3-verified-completion-payload` `DONE` tester — Verified end-to-end: invoked via raw RPC `performance.optimize_shaders mode:"changed" forceRecompile:false` → ticket `j_20260427T024120_c6b47079`. `system.job_status` ~8 s later returned `status:"completed"` with `result:{compiled:true}`. Ticker exited cleanly when `IsCompiling()` cleared.
