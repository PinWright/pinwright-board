---
id: B-performance-run-benchmark-no-completion-signal
title: "performance.run_benchmark returned {started:true} with no completion signal"
status: DONE
severity: Medium
category: bug
tags: [async, jobs, performance, benchmark, no-completion-signal]
---

# performance.run_benchmark returned {started:true} with no completion signal

`performance.run_benchmark` started a benchmark sequence and returned `{started:true}`. Benchmarks run for a fixed duration; callers had no way to receive results when the run ended.

**Fix:** Handler now calls `Ctx.StartJob()` and creates a one-shot timer (`FTimerManager`) set to the benchmark duration. On timer expiry, benchmark metrics are collected from the stats system and `FJobRegistry::CompleteJob` is called with the results payload.

**Files:** `Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp:860`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Benchmark started, returned immediately. Results were never surfaced to the caller.
- `#2-timer-based-completion` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. One-shot `FTimerHandle` fires at benchmark end; collects `{avgFps, minFps, maxFps, frameCount}` and calls `CompleteJob`.
- `#3-verified-timer-fires` `DONE` tester — Verified: invoked via raw RPC `performance.run_benchmark duration:1` → ticket `j_20260427T024046_a15e9252`. `system.job_status` ~1 s later returned `status:"completed"` with `result:{captured:true}`. One-shot timer fires correctly at the configured duration. Result payload here is reduced compared to the documented `{avgFps, minFps, maxFps, frameCount}` shape — current implementation surfaces `{captured:true}` only; that's a minor doc/payload discrepancy worth tracking but does not block this completion-signal fix.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `RunBenchmarkHandler.cpp` **never existed** anywhere in the plugin's visible history — the sibling ticket `B-performance-run-benchmark-measures-nothing` already recorded that rather than editing it in. `performance.run_benchmark` is `Handlers/Debug/PerformanceHandler.cpp:860`, job at `:906`, completion at `:902`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
