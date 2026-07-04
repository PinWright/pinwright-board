---
id: E-stop-profiling-no-uestats-path
title: "performance.stop_profiling returns no .uestats path, forcing the caller to mtime-sort Saved/Profiling/UnrealStats on disk"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [performance, profiling, uestats, result-misreport, docs]
encounters: 3
lastSeen: 2026-07-02T12:09:19.2130448+03:00
---

# `performance.stop_profiling` writes a `.uestats` file but never reports its path

`performance.start_profiling` / `performance.stop_profiling` wrap `stat startfile` /
`stat stopfile` to capture a `.uestats` stat file (the registered descriptions both
say the file "is written under `Saved/Profiling/UnrealStats/`" and the
`start_profiling` blurb adds "load in the Profiler tool"). The capture works — a real
`.uestats` lands on disk — but `stop_profiling` returns only `{"message":"Profiling
stopped"}` with **no path field**. The caller is told a file was written and where the
*directory* is, but never which file, so to actually locate the artifact it just
produced it must fall back to filesystem tooling: `Glob` / mtime-sort
`Saved/Profiling/UnrealStats/*.uestats` and guess the newest is the right one.

This is the same class of result-misreport as the sibling
`E-insights-snapshot-empty-filepath` (insights namespace: snapshot writes a `.utrace`
but reports `filePath:""`), but in the `performance.*` namespace and via a different
mechanism — `stop_profiling` omits the path field entirely rather than echoing an
empty input. The asymmetry against the rest of the profiling surface is the giveaway:
`insights.stop_session` / `insights.get_trace_path` both resolve and return the real
`tracePath`; only the `performance.*` stat-file capture leaves the artifact unreported.

Root cause (`Source/.../Handlers/Debug/PerformanceHandler.cpp`,
`performance.stop_profiling`, :198-199): the handler runs
`GEngine->Exec(..., TEXT("stat stopfile"))` then `Ctx.SendSuccess(TEXT("Profiling
stopped"))` — a bare success message, no result object, no path resolution. The same
shape applies to the `start_profiling` handler (:183-184) and to the inline
`stat startfile` / `stat stopfile` pair driven inside `performance.run_benchmark`
(:759/:765, which returns only `{captured:true}`), so a benchmark run that captures
stats has the same blind spot. (Line numbers shifted from the original ~55/~465/~471
citations after the `E-memory-report-no-path` #4 fix inserted `generate_memory_report`
at :60-171 above them.)

## Process friction this caused (this task)

A technical-artist baseline task ran `start_profiling` → 5s `run_benchmark` →
`stop_profiling` to produce a `.uestats` for the Profiler. Quoting the task's friction
note: *"performance.stop_profiling returned `{"message":"Profiling stopped"}` with NO
written .uestats path, though the wiki/success-check promise it reports one — the file
WAS written (confirmed on disk via Glob), so the response just omits the path."* The
agent's success-check therefore had to leave the MCP surface entirely and stat the
filesystem (`Glob` on `Saved/Profiling/UnrealStats/`, finding the two real files
6.0MB/5.1MB) to confirm the capture, instead of reading a path straight out of the
`stop_profiling` response. Extra off-MCP verification steps for an artifact the tool
itself produced.

**Impact:** an agent that captures a stat file cannot programmatically hand the path
to a downstream step (open in Profiler, attach to a report, diff before/after) from the
response alone — it must mtime-sort the directory, which is racy if any other capture
is in flight. Low severity (the file is written, not lost; a directory mtime-sort
recovers it), but it's pure avoidable friction and it directly contradicts the
descriptions' "load in the Profiler tool" promise.

**Workaround:** after `stop_profiling`, `Glob`/mtime-sort
`Saved/Profiling/UnrealStats/*.uestats` and take the newest.

**Fix:** have `stop_profiling` return the resolved `.uestats` path in a result object
(`{message, statFilePath}`) instead of a bare message — but resolve it **deterministically
from the engine, NOT by mtime-sorting the directory** (the racy "newest file after exec"
approach the sibling fixes `E-memory-report-no-path` #4 and `E-insights-snapshot-empty-filepath`
#3 both had to reject). The stats system records the finalized capture's absolute path in
`FCommandStatsFile::Get().LastFileSaved` (Core `Stats/StatsFile.h`, set in
`IStatsWriteFile::Stop()` at `StatsFile.cpp:276`), and because `stat stopfile` routes through
`UE::Stats::DirectStatsCommand` with `bBlockForCompletion=true`
(`StatsCommand.cpp:2268,2449-2453`), `GEngine->Exec("stat stopfile")` **blocks** on the stats
pipe until that member is set — so reading it straight after the Exec is race-free (the stat
capture is a singleton: `FCommandStatsFile::Start()` calls `Stop()` first, so there is never a
concurrent writer). Gate on `FCommandStatsFile::Get().IsStatFileActive()` **before** stopping so
a stop-without-a-start reports `pathResolved:false` + the `UnrealStats` dir rather than a stale
prior `LastFileSaved`. Apply the same `LastFileSaved` read to the `run_benchmark` inline capture
(its start+stop happen in one job context, so it is trivially in-scope). `start_profiling`
cannot know the engine's timestamped name until stop finalizes it, so it just reports the target
`statsDir`. Guard all uses with `#if STATS`. Mirrors how `insights.stop_session` returns
`tracePath`. Docs follow-up: the `docs/wiki-src/performance.md` overlay (currently a 3-line
intro) should document the returned `statFilePath`/`statsDir` fields once added, so the
discovery story matches the result shape.

## History
- `#4-reword-and-fix` `IN-REVIEW` developer — Reworded the **Fix:** off the racy "newest `.uestats` after `stat stopfile`" mtime-sort (the same correction siblings #4/#3 made) onto a deterministic engine read. Verified in engine source that `stat stopfile` routes through `UE::Stats::DirectStatsCommand(bBlockForCompletion=true)` (`StatsCommand.cpp:2268` → `Task.Wait()` at `:2452`), so `GEngine->Exec("stat stopfile")` blocks on the stats pipe until `IStatsWriteFile::Stop()` sets `FCommandStatsFile::Get().LastFileSaved` to the finalized `<ProfilingDir>/UnrealStats/<name>.uestats` (`StatsFile.cpp:276`) — race-free (stat capture is a singleton). Implemented: `stop_profiling` now gates on `IsStatFileActive()` then returns `{message, statFilePath}` (deterministic `LastFileSaved` read, absolutized); `run_benchmark`'s inline capture returns `statFilePath` too; `start_profiling` returns `{message, statsDir}`; all under `#if STATS` with a `statsDir` + `pathResolved:false` fallback when no capture was active / STATS is compiled out. Corrected stale line cites (start `:183/:184`, stop `:198/:199`, run_benchmark `:759/:765`). Files: `Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp` (handlers + summaries). Test: added `PinWright.performance.stop_profiling.ReturnsResolvedStatFilePath` in `Source/PinWright/Private/Tests/EditorOps/TestDebugHandlers.cpp` — drives the real production handlers (start a capture via `start_profiling`, stop via `stop_profiling`), asserts the stop response carries a non-empty `statFilePath` ending in `.uestats` that exists on disk (cleans up the artifact; guards commandlet/no-GEditor and `!STATS`). Would fail if reverted to the bare `SendSuccess("Profiling stopped")`.
- `#3-liveness` `OPEN` reporter — still reproduces at HEAD
- `#2-additional-repro-confirm` `OPEN` reporter — Replay-confirmed on a fresh perf-baseline task (apply_baseline_settings=performance → set_scalability(1) → frame limit 60 → vsync off → resolution 75% → nanite on → show_fps → start/stop_profiling). `performance.stop_profiling` with `args={}` returned verbatim `{"message":"Profiling stopped"}` — no path field — while `start_profiling` returned `{"message":"Profiling started"}`. The capture works: a real `.uestats` landed on disk at `Saved/Profiling/UnrealStats/ExampleProjectWelcome-WindowsEditor-06.24-09.20.52/Pid42492_...09.22.02.uestats` (4.4MB), so the response simply omits the artifact path. Same off-MCP verification (mtime-sort the dir) was required to confirm. Reaffirms the existing ergonomic finding; no new info on root cause.
- `#1-initial-audit` `OPEN` reporter — `performance.stop_profiling` returns `{"message":"Profiling stopped"}` with no path field despite the registered description promising a `.uestats` "is written under Saved/Profiling/UnrealStats/" and "load in the Profiler tool". Source-confirmed: handler (`PerformanceHandler.cpp`, `stop_profiling`) runs `stat stopfile` then `Ctx.SendSuccess("Profiling stopped")` — no result object, no path resolution; same shape on `start_profiling` (~line 55) and the inline `stat startfile`/`stat stopfile` in `run_benchmark` (~465/471). PROCESS friction (perf-baseline task): the agent's success-check had to leave the MCP and `Glob` `Saved/Profiling/UnrealStats/*.uestats` to confirm the two real files (6.0MB/5.1MB) it had just written, because the response omits the path. Same class as `E-insights-snapshot-empty-filepath` (path-field misreport) but distinct namespace/handler/mechanism (field omitted, not echoed-empty). Workaround: mtime-sort the dir. Fix: return `statFilePath` from a resolver; document on `docs/wiki-src/performance.md`.
