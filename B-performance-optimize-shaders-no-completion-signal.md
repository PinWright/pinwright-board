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

**Files:** `Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp:1075`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Shader compile kicked off, returned immediately with no completion event.
- `#2-poll-gshadercompilingmanager` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Ticker polls `GShaderCompilingManager->IsCompiling()` at 1 s interval until false.
- `#3-verified-completion-payload` `DONE` tester — Verified end-to-end: invoked via raw RPC `performance.optimize_shaders mode:"changed" forceRecompile:false` → ticket `j_20260427T024120_c6b47079`. `system.job_status` ~8 s later returned `status:"completed"` with `result:{compiled:true}`. Ticker exited cleanly when `IsCompiling()` cleared.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `Handlers/Performance/` never existed; the verb is `Handlers/Debug/PerformanceHandler.cpp:1075`, job at `:1120`, completion at `:1114`. The ticker interval is **0.5 s** (`:1119`), not the 1 s the body documents. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
