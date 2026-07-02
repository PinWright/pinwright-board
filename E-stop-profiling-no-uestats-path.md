---
id: E-stop-profiling-no-uestats-path
title: "performance.stop_profiling returns no .uestats path, forcing the caller to mtime-sort Saved/Profiling/UnrealStats on disk"
status: OPEN
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
`performance.stop_profiling`): the handler runs
`GEngine->Exec(..., TEXT("stat stopfile"))` then `Ctx.SendSuccess(TEXT("Profiling
stopped"))` — a bare success message, no result object, no path resolution. The same
shape applies to the `start_profiling` handler (line ~55) and to the inline
`stat startfile` / `stat stopfile` pair driven inside `performance.run_benchmark`
(lines ~465/471), so a benchmark run that captures stats has the same blind spot.

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

**Fix:** have `stop_profiling` resolve the just-written `.uestats` (newest file under
`Saved/Profiling/UnrealStats/` immediately after `stat stopfile`, or capture the name
`stat startfile` chose at start time) and return it as `{statFilePath: "..."}` in a
result object instead of a bare message — mirroring how `insights.stop_session`
returns `tracePath`. Apply the same to the `run_benchmark` inline capture so a
benchmark that records stats reports where they went. Docs follow-up: the
`docs/wiki-src/performance.md` overlay (currently a 3-line intro) should document the
returned `statFilePath` field on `start_profiling`/`stop_profiling`/`run_benchmark`
once added, so the discovery story matches the result shape.

## History
- `#3-liveness` `OPEN` reporter — still reproduces at HEAD
- `#2-additional-repro-confirm` `OPEN` reporter — Replay-confirmed on a fresh perf-baseline task (apply_baseline_settings=performance → set_scalability(1) → frame limit 60 → vsync off → resolution 75% → nanite on → show_fps → start/stop_profiling). `performance.stop_profiling` with `args={}` returned verbatim `{"message":"Profiling stopped"}` — no path field — while `start_profiling` returned `{"message":"Profiling started"}`. The capture works: a real `.uestats` landed on disk at `Saved/Profiling/UnrealStats/ExampleProjectWelcome-WindowsEditor-06.24-09.20.52/Pid42492_...09.22.02.uestats` (4.4MB), so the response simply omits the artifact path. Same off-MCP verification (mtime-sort the dir) was required to confirm. Reaffirms the existing ergonomic finding; no new info on root cause.
- `#1-initial-audit` `OPEN` reporter — `performance.stop_profiling` returns `{"message":"Profiling stopped"}` with no path field despite the registered description promising a `.uestats` "is written under Saved/Profiling/UnrealStats/" and "load in the Profiler tool". Source-confirmed: handler (`PerformanceHandler.cpp`, `stop_profiling`) runs `stat stopfile` then `Ctx.SendSuccess("Profiling stopped")` — no result object, no path resolution; same shape on `start_profiling` (~line 55) and the inline `stat startfile`/`stat stopfile` in `run_benchmark` (~465/471). PROCESS friction (perf-baseline task): the agent's success-check had to leave the MCP and `Glob` `Saved/Profiling/UnrealStats/*.uestats` to confirm the two real files (6.0MB/5.1MB) it had just written, because the response omits the path. Same class as `E-insights-snapshot-empty-filepath` (path-field misreport) but distinct namespace/handler/mechanism (field omitted, not echoed-empty). Workaround: mtime-sort the dir. Fix: return `statFilePath` from a resolver; document on `docs/wiki-src/performance.md`.
