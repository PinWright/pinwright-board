---
id: F-insights-export-trace
title: "insights.export_trace — dump timer/frame/counter stats from a .utrace via TraceServices"
status: DONE
severity: Medium
category: feature
tags: [insights, profiling, trace, traceservices, async, jobs, csv]
---

# insights.export_trace — dump timer/frame/counter stats from a .utrace via TraceServices

The `insights.*` namespace is **capture-only**. `insights.start_session` /
`stop_session` / `get_trace_path` / `snapshot` / `set_channels` let an agent
drive a trace and locate the resulting `.utrace`, but there is no MCP-native
way to get *numbers* back out of that file. Once the trace is written the
agent is stuck: the data is locked inside a binary `.utrace` and nothing in
the plugin reads it.

The two obvious in-engine paths to crack the file open are both dead ends:

- **Headless UnrealInsights CLI export (UE 5.7) is broken.** The documented
  approach — `UnrealInsights.exe -OpenTraceFile=<path>
  -ExecOnAnalysisCompleteCmd="..." -AutoQuit` — does not work on the 5.7
  branch. The Insights frontend respawns the analysis as a *child* process
  and strips `-ExecOnAnalysisCompleteCmd` when it does, so the completion
  command never runs against the session that actually has the analysis
  loaded. `-InsightsTest` did not help either. This is a known open
  regression in the 5.7 headless-export path, not a misconfiguration on our
  side.
- **Unreal's built-in Python cannot reach TraceServices.** The
  `TraceServices` analysis API (`ITraceServices`, `IAnalysisSession`, the
  `Read*Provider` accessors) is **unreflected** — there are no `UCLASS`/
  `UFUNCTION` wrappers, so `unreal.*` Python has no binding to it. There is
  no `python.execute` workaround that queries the providers directly.

So the data exists on disk, the engine ships the library to read it
(`TraceServices`), but neither of the two normal "ask the engine to read it
for me" surfaces can reach that library from an agent.

**Feature:** add `insights.export_trace` — a handler that opens a `.utrace`
through the in-process `TraceServices` analysis API and dumps the headline
profiling tables to CSV for external analysis. Four artifact families:

- **timer_stats** — per-timer aggregated stats (call count, inclusive /
  exclusive time, min/max/avg) across the whole session.
- **frame_series** — per-frame timings, one row per frame.
- **counters** — trace counter (stat) series over time.
- **timers** — the timer/spec table itself (id ↔ name mapping) so the other
  exports are joinable.

Runs as an **async job** (analyzing a multi-hundred-MB trace blocks for
seconds and must not stall the game thread). CSVs land under
`.editor-automation/insights/` (sibling of the existing
`.editor-automation/jobs.jsonl` / `asset-dumps/` trees), one file per family,
so the agent reads them straight off disk and analyzes them in external
pandas — no need to round-trip large tables back through the `tools/call`
channel.

**Implementation approach** (sketch — no compile, no commit):

- Open the file via the trace store / `FTraceAuxiliary` analysis entry point
  to get an `IAnalysisSession`, then read providers off it:
  - **timer_stats** — `TraceServices::ReadTimingProfilerProvider(Session)`,
    then `CreateAggregation(...)` over the session time range to get the
    per-timer aggregated rows (count, incl/excl, min/max/avg).
  - **frame_series** — `ReadFrameProvider(Session)` for the frame list, then
    a **per-thread timeline walk** to attribute work into each frame and
    emit per-frame `busy_ms` / `span_ms` columns.
  - **counters** — `ReadCounterProvider(Session)`, enumerate counters and
    emit their value series.
  - **timers** — the timer/spec table from the timing-profiler provider for
    the id↔name join key.
- **Build.cs** adds the `TraceServices` module dependency (only
  `Core`/`TraceLog`/`TraceAnalysis` are linked today; the analysis-provider
  API lives in `TraceServices`).
- **Async wiring** reuses the existing job infrastructure: `Ctx.StartJob()`
  to register the ticket and send the kickoff response, then
  `Async(EAsyncExecution::Thread, ...)` to run the analysis off the game
  thread, finishing with `FJobRegistry::Get().CompleteJob(JobId, ...)` (the
  same StartJob → Async(Thread) → Complete shape the other long-running
  handlers use). Result payload reports the written CSV paths under
  `.editor-automation/insights/`.

**Workaround:** open the `.utrace` in the **GUI** Unreal Insights manually
and read the numbers off the timing/frames/counters panels by hand. No
headless / scripted path exists until this ships.

## History
- `#1-feature-request` `OPEN` reporter — `insights.*` is capture-only; no MCP path extracts numbers from a written `.utrace`. The two in-engine routes are both dead: (a) UE 5.7 headless UnrealInsights CLI export is a known open regression — the frontend respawns the analysis child and strips `-ExecOnAnalysisCompleteCmd`, and `-InsightsTest` did not help; (b) Unreal built-in Python cannot reach `TraceServices` because the analysis API is unreflected (no UCLASS/UFUNCTION bindings). Proposed `insights.export_trace`: open the trace via `TraceServices`, dump timer_stats (`ReadTimingProfilerProvider` + `CreateAggregation`), frame_series (`ReadFrameProvider` + per-thread timeline walk → per-frame `busy_ms`/`span_ms`), counters (`ReadCounterProvider`), and the timer table; run as an async job (`StartJob` → `Async(EAsyncExecution::Thread)` → `FJobRegistry::Complete`); write CSV under `.editor-automation/insights/` for external pandas analysis; `Build.cs` adds the `TraceServices` dependency. Workaround until then: open the `.utrace` in the GUI Insights by hand.
- `#2-implemented-export-trace` `IN-REVIEW` developer — Implemented `insights.export_trace` in `Handlers/Debug/TraceAnalysisHandler.cpp` (param resolution on the game thread) + `Handlers/Debug/TraceExportCore.{h,cpp}` (UObject-free TraceServices analysis core run on a worker thread via `Ctx.StartJob` → `Async(EAsyncExecution::Thread)`). Build.cs already links `TraceServices`. Emits four kinds — timer_stats (`CreateAggregation` per window, sorted by total inclusive time, capped by `tableEntryLimit`/`topN` with non-silent truncation events), frame_series (per-frame per-thread `busy_ms`/`span_ms` via a depth-0 timeline walk boundary-split into frames), counters (`EnumerateCounters` with optional `counterDownsampleHz`), and timers/threads dictionaries. All provider reads happen under one `FAnalysisSessionReadScope`. Windows are seconds-based; `tracePath` is optional (defaults to newest `.utrace` in the UnrealTrace store / ProfilingDir); default kind is frame_series only. Wiki page authored at `wiki-src/insights.md`. Not yet compiled/verified — awaiting test.
- `#3-verify-fix` `DONE` tester — Verified: ran `insights.export_trace` with `tracePath=Saved/Profiling/20260313_120449.utrace`, `kind=[frame_series,timers,counters]`. Response returned 4 written CSV paths plus summary `{durationSeconds:477.36, frameCount:11454, threadCount:80}` and `truncation.truncated:false`. Confirmed on disk: `frame_series.csv` has per-frame per-thread `<thread>_busy_ms`/`<thread>_span_ms` columns (GameThread/RenderThread 0/AudioMixerRenderThread/RHIThread) with real values; `timers.csv` maps timer_id↔name with type/file/line. TraceServices analysis end-to-end works.
