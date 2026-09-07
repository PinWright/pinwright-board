---
id: F-performance-frame-time-statistics
title: "No verb in the performance.* namespace returns a measured performance quantity: all 19 either mutate a render CVar, echo the caller's input, toggle a viewport overlay or write a file, and the two that claim to profile fail NOT_SUPPORTED on UE 5.8 while run_benchmark returns {captured:false} — so a whole vegetation profiling session had to be measured with raw `CsvProfile FRAMES=150` and `ProfileGPU` console strings whose output console_command does not even capture, then parsed off disk by hand"
status: OPEN
severity: Medium
category: feature
tags: [performance, profiling, benchmark, frame-time, fps, gputime, measurement, readback, missing-verb, csvprofile, profilegpu, insights, run_benchmark, stat-unit, percentiles]
encounters: 1
costly: 1
lastSeen: 2026-08-30T17:40:00+03:00
---

# A namespace for performance that cannot report performance

`performance.*` has **19 registered verbs**. Every one of them either sets a render CVar, echoes
back what the caller passed, toggles a viewport overlay, merges actors, or writes a file. **Not one
returns a measured quantity** — no frame time, no FPS, no ms, no GPU time, no draw calls, no
primitives drawn.

Six of the nineteen *do* return numbers, and the distinction matters enough to state precisely
rather than overclaim: `set_scalability` returns `requestedLevel` plus eleven `appliedGroups[].value`
(`PerformanceHandler.cpp:453`), `apply_baseline_settings` seven `appliedCVars[].value` (`:997`),
`merge_actors` three counts (`:840-855`), `configure_occlusion_culling` `slop` and
`minScreenRadius` (`:1070`), and `enable_gpu_timing` / `optimize_draw_calls` booleans (`:934`,
`:1023`). **Every one of those numbers is either the caller's own input echoed back or an
`IConsoleVariable::GetInt()` read of a setting.** None was measured from a running frame.

## The roster, re-derived at HEAD

`Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp` (1122 lines), one
`REGISTER_RPC_HANDLER` each. Count cross-checked three ways: 19 registration lines in the file,
19 `performance.*` method pages under `Saved/PinWright/wiki/`, and 19 rows from
`grep -c '^performance\.'` over the generated verb registry.

| verb | reg. line | what the response carries |
|---|---|---|
| `generate_memory_report` | `:77` | `message`, `path` (send `:186`) |
| `start_profiling` | `:252` | `message`, `statsDir` (`:277`) — **or `NOT_SUPPORTED`, `:255-260`** |
| `stop_profiling` | `:283` | `message`, `statFilePath` (`:307`) — **or `NOT_SUPPORTED`, `:289`** |
| `show_fps` | `:323` | fixed string `"FPS stat toggled"` (`:334-335`) |
| `show_stats` | `:340` | fixed string (`:380`) |
| `set_scalability` | `:396` | requested level + per-group CVar readback (`:453`) |
| `set_resolution_scale` | `:458` | fixed string (`:474`) |
| `set_vsync` | `:479` | fixed string (`:490`) |
| `set_frame_rate_limit` | `:495` | fixed string (`:508`) |
| `configure_nanite` | `:513` | fixed string (`:524`) |
| `configure_lod` | `:529` | fixed string (`:553`) |
| `configure_texture_streaming` | `:558` | fixed string (`:593`) |
| `merge_actors` | `:606` | actor/component counts, package paths (`:840-855`) |
| `run_benchmark` | `:860` | job ticket; result `{captured:bool}` + optional `statFilePath` (`:894-900`) |
| `enable_gpu_timing` | `:911` | `{enabled:bool}` (`:934`) |
| `apply_baseline_settings` | `:939` | `appliedCVars[]` (`:997`) |
| `optimize_draw_calls` | `:1002` | `{optimized, instancing}` (`:1023`) |
| `configure_occlusion_culling` | `:1028` | input echoes (`:1070`) |
| `optimize_shaders` | `:1075` | job ticket; `{compiled:true}` (`:1120`) |

**Nothing in the plugin reads a frame time.** Swept `Source/` for the engine's own measurement
APIs — `GAverageFPS`, `GAverageMS`, `GGPUFrameTime`, `RHIGetGPUFrameCycles`, `GetAverageFrameTime`,
`SmoothedFrameRate`, `FCsvProfiler`, `FPerformanceTrackingSystem`, `IPerformanceDataConsumer`,
`FStatsThreadState`, `FComplexStatMessage` — **zero hits**. Likewise zero
`SetNumberField(TEXT("fps"|"frameTime"|"gpuTime"|...))`. The only `FPlatformTime::Seconds()` uses
are RPC wall-clock `durationMs` (e.g. `Dispatch/RpcDispatcher.cpp:757`), which times the call, not
the frame.

## The three verbs that look like the answer, and are not

**`show_fps` and `show_stats` are pure overlays with no readback.** `:334` is
`GEngine->Exec(World, TEXT("stat fps"))` and `:335` is `SendSuccess(TEXT("FPS stat toggled"))`.
The number is drawn on a viewport the caller cannot read, and the response does not even report
which way the toggle went.

**`start_profiling` / `stop_profiling` fail loud on this engine.** `PerformanceHandler.cpp:27-31`
defines `PINWRIGHT_HAS_STATS_FILE_CAPTURE` from `UE_ENABLE_STATS_FILE_DEPRECATED_IN_5_8`, which
`C:/UE_5.8/Engine/Source/Runtime/Core/Public/Stats/StatsFile.h:8-9` defaults to **0**. Both verbs
therefore take the `#if !` branch — `:255-260` and `:289` — and return `NOT_SUPPORTED`, correctly
refusing to fabricate a capture. That is honest behaviour and not a defect; it does mean the
`.uestats` route is simply gone on UE 5.8, and it was never machine-readable anyway (the artefact is
for the standalone Profiler GUI).

**`run_benchmark` is the one that should worry a reader of this ticket.** Its registered summary at
`:860` is the whole of *"Start a performance benchmark"*. Its body (`:860-907`) takes a `duration`
(default 5) and an ignored `type`, issues `stat startfile`, waits on an `FTSTicker`, and finalizes.
On UE 5.8 the capture is compiled out by the same macro, so the result at `:894-900` is
`{captured: false}` and nothing else — **reported as a successful job**, not as `NOT_SUPPORTED`
like its two siblings. Its own comment at `:891-893` states the reasoning. So the verb whose name
promises a measurement returns a boolean saying no measurement happened, in a response the caller
must inspect to notice. `B-performance-run-benchmark-no-completion-signal` (DONE, Medium) recorded
this in its `#3` as *"a minor doc/payload discrepancy worth tracking"* and nothing tracked it; that
ticket also mentions a documented payload `{avgFps, minFps, maxFps, frameCount}` — **which does not
exist**: `avgFps` / `minFps` / `maxFps` appear nowhere in `Plugins/PinWright/`, not in source, not
in `Docs/wiki-src/`, not in the generated wiki.

## PinWright *can* answer the question — in a different namespace, offline, and nobody used it

Scoping this honestly matters, because "PinWright cannot measure frame time" is refutable in one
call. `insights.export_trace` (`Handlers/Debug/TraceAnalysisHandler.cpp:138`, core in
`TraceExportCore.cpp`) does return timing:

- over the wire, `TraceExportCore.cpp:724-726` — `summary.durationSeconds`, `summary.frameCount`,
  `summary.threadCount`, i.e. an average FPS is derivable;
- to disk, `frame_series.csv` (`TraceExportCore.cpp:341-460`) with
  `frame_index,start_time,end_time,wall_ms,<Thread>_busy_ms,<Thread>_span_ms` for GameThread /
  RenderThread / RHIThread, and `timer_stats__<window>.csv` (`:294-305`) with total / average /
  min / max / median inclusive and exclusive ms per timer.

But it is post-hoc: it needs a `.utrace`, a session start and stop, an export, and then a CSV
parse. It answers "what did that recording cost", not "how fast is it right now". And it was never
reached for during the session that produced this ticket — `Saved/PinWright/insights/` does not
exist in this checkout and no `frame_series.csv` / `timer_stats*.csv` was ever written.

**So the claim this ticket makes is scoped to the namespace: `performance.*` has no live
measurement verb, and the only timing path in the plugin is an offline trace export in
`insights.*`.**

## What the session actually did instead — and PinWright's own docs prescribe it

A full vegetation performance pass (`Docs/map/vegetation-performance.md` on host
`EAContentExamples58`, UE 5.8, editor build 13:32 / plugin `d8f1bc32`) measured every number through
raw console strings and hand parsing:

- `CsvProfile FRAMES=150` -> `Saved/Profiling/CSV/*.csv`, read by a hand-written stdlib parser
  (`dev/perf/csv_stats.py`, header: *"Host-side reader for Saved/Profiling/CSV/*.csv produced by
  `CsvProfile FRAMES=N`"*); five `Profile(*).csv` files on disk.
- `ProfileGPU` scraped out of `Saved/Logs/EAContentExamples58.log` by `dev/perf/gpu_profile_parse.py`.

**And the console verb does not return the command's output**, so even the raw route is half blind:
`editor.console_command` (`Handlers/Editor/EditorCommandHandler.cpp:289`) returns
`{success, command, world, worldPath, consumed, message}` and no Exec output —
`CsvProfile FRAMES=150` through PinWright returns literally nothing, and the answer has to be found
on disk afterwards. The machinery to fix that already exists and is used once:
`FMcpOutputCapture : public FOutputDevice` at `Utils/LogUtils.h:11`, used by
`Handlers/Actor/LifecycleHandler.cpp:401` and the tests. Nothing scrapes `stat unit`.

The plugin's own documentation already sends callers down this road, which is the clearest evidence
the typed answer is missing: `Docs/wiki-src/insights.tracing-reference.md:79` prescribes `CsvProfile`
via `editor.console_command`, and `Docs/wiki-src/insights.stat-companions.md:11,14` describes
`stat unit` as *"per-frame ms"* with `performance.show_stats` as *"the toggle wrapper"*. A wrapper
around an overlay is the whole of what the namespace offers.

## Ask

**A verb that samples N frames and returns the distribution.** Shape, offered as a starting point
rather than a specification:

```
performance.measure_frames { frames: 150, warmupFrames: 30 }
->
{
  frames: 150, warmupFrames: 30, droppedFrames: 0,
  frameTimeMs:  { p10, p50, p90, p99, min, max, mean },
  gameThreadMs: { ... }, renderThreadMs: { ... }, rhiThreadMs: { ... }, gpuMs: { ... },
  drawCalls: { ... }, primitivesDrawn: { ... },
  renderResolution: { width, height, screenPercentage },
  viewport: { width, height }, realtime: true
}
```

Four properties that are not decoration:

1. **Percentiles, not a mean.** The measured behaviour on this host is that identical draw calls
   intermittently triple `GPUTime` (one capture: p25 15.08, p50 20.29, p75 41.24 ms). A stall can
   only add time, so the low percentile tracks the real cost — p10 reproduced to 0.07 ms across
   independent samples where the median wandered by 4 ms. A verb returning only a mean or a median
   would be worse than nothing.
2. **The thread split.** "22 ms" without knowing whether it is game, render, RHI or GPU does not
   tell a caller what to change; the session's whole conclusion (GPU-bound in the renderer, not in
   content) came from that split.
3. **`primitivesDrawn` / `drawCalls` as the same-scene guard.** In a shared editor it is the
   cheapest way to notice that someone changed the level between two halves of a comparison — it
   jumped 95,648 -> 151,568 mid-comparison once in this session.
4. **The render resolution, from `E-viewport-info-no-render-resolution`** (OPEN, Medium). That
   ticket's ask item 3 is *"include the render resolution in the profiling receipts — whatever
   `performance.run_benchmark` and the CSV/`ProfileGPU` capture verbs return"*, which presupposes
   receipts this ticket says do not exist. **The two are complements: that ticket wants the
   denominator, this one wants the numerator, and a frame time without a resolution is not a
   result.** Whichever lands second should carry both fields.

A smaller, cheaper alternative that would still close most of the gap: **capture `stat unit` /
`CsvProfile` output on the console verbs** using the existing `FMcpOutputCapture`
(`Utils/LogUtils.h:11`), so the raw route at least returns its own answer instead of leaving it on
disk.

## Distinct from

- **`E-viewport-info-no-render-resolution`** (OPEN, Medium) — the denominator, as argued above.
  Neither closes the other.
- **`B-performance-run-benchmark-no-completion-signal`** (DONE, Medium) — fixed the *completion
  signal* (a job ticket). Its `#3` flagged the empty payload as worth tracking; this ticket is where
  that thread is picked up, and the `{avgFps, minFps, maxFps, frameCount}` shape it calls
  "documented" is re-derived here as never having existed in source or docs.
- **`B-inspect-settings-stats-stub-silent-success`** (IN-REVIEW, Medium) —
  `system.inspect.get_performance_stats` / `get_memory_stats` are stubs returning
  `{"success":true,"message":"Performance stats placeholder - implement with actual metrics"}`.
  That is the *stub* defect (a verb lying about being implemented); this is the *capability* gap in
  a different namespace. If that stub is ever implemented into a real frame-time readback the two
  should merge, and whoever does should read this ticket's § *Ask* first.
- **`F-insights-export-trace`** (DONE, Medium) — shipped the offline CSV pipeline described above.
  It is the reason this ticket is scoped to the namespace instead of the plugin.
- **`E-stop-profiling-no-uestats-path`** (IN-REVIEW, Low) and **`E-memory-report-no-path`**
  (IN-REVIEW, Medium) — return the *path* of an artefact. File locations, not numbers.
- **`E-perf-wp-configure-readback-thin`** (OPEN, Low) and
  **`E-performance-run-benchmark-async-poll-undocumented`** (OPEN, Low) — CVar echo and docs.

Board-wide sweep for `frame time|frametime|fps|profil|benchmark|csvprofile|stat unit|GPUTime|
measure` found nothing asking for a measured frame time in an RPC response. **Unowned.**

## Severity

**Medium**, on the rubric's soft-blocker band: *"doable, but only via a documented workaround, a
source dive, or many extra calls"*. All three apply literally — the workaround is documented in the
plugin's own wiki (`insights.tracing-reference.md:79`), the CSV column semantics came from a source
dive, and it costs a console call plus a file read plus a hand-written parser per measurement.

**Not High.** No verb returns a false value here; `start_profiling` / `stop_profiling` refuse
honestly, and the numbers `set_scalability` and `apply_baseline_settings` return are true. A
reasonable task is not impossible: `insights.export_trace` reaches the answer. (The one thing in
this area that *does* look High is `run_benchmark` returning `{captured:false}` under a success —
that is a silent false-success on the only supported engine, and it is deliberately left as
evidence here rather than rated, because it is a bug in an existing verb and not this feature gap.
It is flagged in `§ The three verbs that look like the answer` for whoever wants to file it.)

**Reach modifier declined.** Profiling is not an every-session path, which by the rubric argues a
bump down to Low. Declined: Low is *"pure friction — docs, discoverability, naming, cosmetic"*, and
this is not friction. The namespace exists to change performance; being unable to observe the thing
it changes is a capability gap, and its cost is not friction but wrong conclusions — the same
session recorded three A/B results (`-27%`, `-16%`, `-20%`) that did not survive re-measurement,
precisely because the measurement had to be assembled by hand outside the tool.

severity rationale: impact=soft blocker, workaround documented but out-of-band and hand-parsed
(Medium) x reach=rare path, bump down to Low declined because a capability gap is not friction
-> Medium

## Not done

Plugin source was not modified and no new measurement was taken for this filing — the roster,
response shapes and greps are read at HEAD, and the session evidence is cited from
`Docs/map/vegetation-performance.md` on the host that produced it. `insights.export_trace` was not
run: establishing that it *can* answer the question is a source-level claim here, and whether its
`frame_series.csv` is a usable substitute in practice is left open.

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
