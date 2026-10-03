---
id: F-performance-frame-time-statistics
title: "performance.run_benchmark measures whole-frame time only: add the game/render/RHI/GPU split, a low percentile, warmup, a draw-call/primitive same-scene guard and the render resolution to its receipt"
status: OPEN
severity: Low
category: feature
tags: [performance, profiling, benchmark, frame-time, fps, gputime, measurement, readback, missing-verb, csvprofile, profilegpu, insights, run_benchmark, stat-unit, percentiles]
encounters: 1
costly: 1
lastSeen: 2026-08-30T17:40:00+03:00
rice: [1, 1, 1, 2]
priority: 4
---

# `run_benchmark` reports one number per frame; a performance comparison needs five more

`performance.run_benchmark` (`Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp:981`,
only param `duration` `:983`) now measures: it samples `FTSTicker` `DeltaTime` every frame
(`:1024-1028`) and returns `frameCount`, `measuredDurationSeconds`, `avgFps`,
`frameTimeMs {min, p50, p95, max, mean}` (`:946-977`) and `backgroundThrottle`, failing with
`FRAME_TIME_NOT_MEASURED` rather than returning no numbers (`:1063`).

That is whole-frame wall time only. A grep of `Source/` for `GGameThreadTime`, `GRenderThreadTime`,
`GRHIThreadTime`, `GGPUFrameTime`, `RHIGetGPUFrameCycles`, `GNumDrawCallsRHI`,
`GNumPrimitivesDrawnRHI` and `FCsvProfiler` returns nothing, so the receipt cannot say:

1. **Which thread is the bottleneck.** "22 ms" without the game/render/RHI/GPU split does not tell a
   caller what to change; the reporting session's conclusion (GPU-bound in the renderer) came from
   that split, taken via `CsvProfile` and `ProfileGPU` off disk.
2. **The stable low percentile.** On the reporting host identical draw calls intermittently tripled
   GPU time (p25 15.08 / p50 20.29 / p75 41.24 ms); p10 reproduced within 0.07 ms where the median
   wandered 4 ms. `min` is a single outlier, `p50` is the unstable one.
3. **Warmup.** The window starts at the call; the first frames after a level or setting change are
   counted.
4. **Same-scene guard.** `drawCalls` / `primitivesDrawn` catch a level edited mid-comparison in a
   shared editor (seen once: 95,648 -> 151,568).
5. **Render resolution.** A frame time without the resolution it was rendered at is not comparable
   across sessions (see `E-viewport-info-no-render-resolution`).

**Workaround:** `insights.start_session` / `insights.export_trace` writes `frame_series.csv` with
per-frame GameThread/RenderThread/RHIThread busy ms; or `CsvProfile FRAMES=N` / `ProfileGPU` via
`editor.console_command` (which returns no command output) and parse `Saved/Profiling/CSV` or the
editor log.

**Fix:** extend `run_benchmark`: optional `warmupFrames` (default 0, unchanged behaviour), per-frame
samples of game/render/RHI thread time and GPU time summarised with the same block shape plus `p10`,
per-frame `drawCalls` / `primitivesDrawn` summaries, and a `renderResolution {width, height,
screenPercentage}` read once at the end (sharing whatever `E-viewport-info-no-render-resolution`
adds). Fields that cannot be measured on the current RHI are omitted with a warning, not zeroed.

**Acceptance:** a `run_benchmark {duration:5, warmupFrames:30}` job result carries `frameTimeMs.p10`,
`gameThreadMs`, `renderThreadMs`, `rhiThreadMs`, `gpuMs`, `drawCalls`, `primitivesDrawn` and
`renderResolution`; on a static scene the `gpuMs.p50` is within a few percent of `stat unit`'s GPU
figure, and `frameCount` excludes the warmup frames.

severity rationale: impact=Medium (the readback omits fields and forces an out-of-band fallback) x
reach=rare path (profiling sessions only), bump down -> Low

## Related

- `B-performance-run-benchmark-measures-nothing` (IN-REVIEW) — made `run_benchmark` measure frame
  time; this is the residual.
- `E-viewport-info-no-render-resolution` (OPEN) — the render size on `get_viewport_info`.
- `F-insights-export-trace` (DONE) — the offline thread-split route used as the workaround.

## History
- `#1-performance-namespace-cannot-measure` `OPEN` reporter — Filed from the final triage sweep of a
  vegetation performance session on host `EAContentExamples58` (UE 5.8, editor build 13:32, plugin
  commit `d8f1bc32`). Roster re-derived at HEAD: 19 `REGISTER_RPC_HANDLER("performance.*")` lines in
  `Handlers/Debug/PerformanceHandler.cpp`, cross-checked against 19 wiki method pages and 19 registry
  rows. **The lead I was handed said "none of the 19 returns a number" and that did not survive** —
  six do (`:453`, `:997`, `:840-855`, `:1070`, `:934`, `:1023`) — so the claim is restated as *none
  returns a measured quantity*, every one of those numbers being an input echo or an
  `IConsoleVariable::GetInt()`. Swept `Source/` for `GAverageFPS`, `GAverageMS`, `GGPUFrameTime`,
  `RHIGetGPUFrameCycles`, `GetAverageFrameTime`, `SmoothedFrameRate`, `FCsvProfiler`,
  `FPerformanceTrackingSystem`, `IPerformanceDataConsumer`, `FStatsThreadState`,
  `FComplexStatMessage`: zero hits. **Two more lead claims corrected:** `start_profiling` /
  `stop_profiling` do not "emit a `.uestats`" on this engine — they return `NOT_SUPPORTED`
  (`:255-260`, `:289`) because `PINWRIGHT_HAS_STATS_FILE_CAPTURE` (`:27-31`) resolves through
  `UE_ENABLE_STATS_FILE_DEPRECATED_IN_5_8`, which `C:/UE_5.8/.../Stats/StatsFile.h:8-9` defaults to
  0; and "PinWright cannot measure frame time" is too strong — `insights.export_trace`
  (`TraceAnalysisHandler.cpp:138`) returns `summary.durationSeconds` / `frameCount` at
  `TraceExportCore.cpp:724-726` and writes per-frame `frame_series.csv` (`:341-460`) and
  `timer_stats__*.csv` (`:294-305`), so the claim is scoped to the namespace. Two findings not in the
  lead: `run_benchmark` (`:860`) returns `{captured:false}` under a *successful* job on UE 5.8
  (`:894-900`, comment `:891-893`) rather than refusing like its siblings, and the
  `{avgFps,minFps,maxFps,frameCount}` payload `B-performance-run-benchmark-no-completion-signal` `#3`
  calls documented exists nowhere in the plugin; and `editor.console_command`
  (`Handlers/Editor/EditorCommandHandler.cpp:289`) captures no Exec output, so even the raw
  `CsvProfile` route returns nothing, although `FMcpOutputCapture` (`Utils/LogUtils.h:11`) already
  exists and is used at `Handlers/Actor/LifecycleHandler.cpp:401`. Ask: a `performance.measure_frames`
  returning p10/p50/p90/p99 split game/render/RHI/GPU plus `drawCalls`/`primitivesDrawn` and the
  render resolution, with percentiles argued as load-bearing (measured p25 15.08 / p50 20.29 / p75
  41.24 ms on identical draw calls) and cross-linked to `E-viewport-info-no-render-resolution`, whose
  ask item 3 presupposes receipts this ticket shows do not exist. Severity Medium on the soft-blocker
  band; High declined because nothing returns a false value and `insights.export_trace` reaches the
  answer; reach bump down to Low declined because a capability gap is not friction and its measured
  cost was three A/B conclusions that did not survive re-measurement.
- `#2-rephrased` `OPEN` developer — Old headline ("no `performance.*` verb returns a measured quantity", `run_benchmark` returns `{captured:false}`) is false at 7230b41d: `run_benchmark` samples every frame and returns `frameCount`, `avgFps` and `frameTimeMs {min,p50,p95,max,mean}` (`PerformanceHandler.cpp:946-977`, `:1024-1028`; `B-performance-run-benchmark-measures-nothing`). Retitled and rewritten to the residual: thread/GPU split, p10, warmup, draw-call/primitive guard and render resolution on the `run_benchmark` receipt (render resolution moved here from `E-viewport-info-no-render-resolution`); dropped the 19-verb roster, the `{captured:false}` and `start/stop_profiling` sections and the `measure_frames` new-verb proposal, added Acceptance. Severity Medium->Low: the core measurement exists, the remainder is a readback gap with an `insights.export_trace` fallback on a rare path.
