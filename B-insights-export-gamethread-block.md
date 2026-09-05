---
id: B-insights-export-gamethread-block
title: "`insights.export_trace` parses and exports an arbitrarily large trace synchronously on the game thread despite advertising a worker job"
status: IN-REVIEW
severity: High
category: bug
tags: [insights, trace, game-thread, unbounded-wait, documentation]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Trace export monopolizes the game thread with no job or bound

## What happens

The registration says `insights.export_trace` loads on a worker thread and returns
a job ticket (`TraceAnalysisHandler.cpp:138-150`). The implementation instead calls
`RunTraceExport` synchronously on the game thread and sends the final response
inline (`:305-316`). The core blocks in `AnalysisService->Analyze` until the whole
trace is parsed (`TraceExportCore.cpp:611-618`) and then scans providers and builds
all requested CSV contents before returning (`:631-710`). There is no timeout,
progress, cancellation or size bound.

The source comment assumes a typical trace takes about one second, but the caller
controls `tracePath`; trace size and provider work are unbounded.

## Why it matters

A large or pathological trace can freeze editor UI and starve the RPC transport for
the complete parse/export. The documented job contract also causes callers to
expect pollable progress that never exists. Severity is High for an unbounded
game-thread operation without a reproduced editor death.

## What should happen

Restore a real bounded job contract. Because the comment records that some analyzers
assert game-thread entry, do not blindly move `Analyze` to a worker. Use a bounded,
incremental game-thread state machine, an isolated analyzer process, or explicitly
limit unsupported analyzers; run UObject-free CSV work off-thread, and provide
progress, timeout and cancellation. Make the registration match the delivered
contract.

## Workaround

Use Unreal Insights externally for large traces. Limit this verb to known-small
traces while the editor is otherwise idle.

## Related

- `F-insights-export-trace` — feature ticket describes the former job design but
  does not track the current synchronous implementation.

## Fix

**Verdict: TRUE.** The handler registration described a worker/job contract, but the
handler called `RunTraceExport` inline, so blocking `Analyze`, provider scans, and
CSV construction occupied the game-thread stack. The first fix also needed the
engine-supported non-blocking analysis split because analyzer startup has game-
thread affinity.

**Changed files:**

- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Debug\TraceAnalysisHandler.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Debug\TraceExportCore.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Debug\TraceExportCore.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\ErrorCodes.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\InsightsHandlerTest.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Debug\InsightsHandlerInternal.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\wiki-src\insights.md`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\wiki-src\performance-profiling.headless-insights.md`

**Fix:** `FTraceExportJob` uses `PinWrightSafePoint::DeferJobToSafePoint` for the
retained active-request continuation, starts `TraceServices::StartAnalysis` on
the game thread, and then lets the worker poll `IAnalysisSession::IsAnalysisComplete`.
The worker performs provider reads and CSV export, stops/waits for/releases the
analysis session off-thread, and only then returns through `DeferToSafePoint`.
`timeoutSeconds` is bounded (300-second default, 1800-second maximum); cancellation
and dispatcher abandonment share owned atomic state, exactly-once cleanup, and no
nested game-thread marshal. A process-local canonical output-directory reservation
rejects concurrent requests targeting the same artifacts.
Deadline enforcement is cooperative around plugin-controlled enumeration boundaries; an individual engine aggregation or OS write is not forcibly preempted.

**Test IDs:** `PinWright.insights.export_trace.WriteFailureIsTypedAndLeavesNoPartialFile`,
`PinWright.insights.export_trace.NoGameFramesAreTypedFailure`, and
`PinWright.insights.export_trace.TimeoutIsTypedAndLeavesNoArtifacts` (handler
harness; static-only verification in this pass).

**Deliberately unchanged:** TraceServices analysis semantics, CSV schemas and
requested-kind selection remain unchanged; no PIE/live-editor run or progress
reporting was added. Large-trace timing and analyzer runtime behavior remain
follow-up verification because this pass was static-only.

## History
- `#1-source-scan-synchronous-export` `OPEN` reporter — Source-only scan confirmed the handler's inline game-thread call, blocking `Analyze`, whole-provider scans, and immediate final response, contradicting the registered worker/job contract. No build, test, editor, MCP call, or plugin edit was performed.
- `#2-background-safe-point-job` `IN-REVIEW` developer — Moved export work to the background job path with a retained dispatcher continuation and safe-point completion; added the behavioural handler test and updated the Insights overlays. No live runtime verification was performed.
- `#3-analysis-start-split-and-abandonment` `IN-REVIEW` developer — Corrected the partial fix by starting TraceServices analysis at a safe point, polling completion on the worker, suppressing late completion after dispatcher abandonment, and failing missing `frame_series` artifacts. Added the no-Game-frames handler test. No live runtime verification was performed.
- `#4-worker-session-teardown` `IN-REVIEW` developer — Moved TraceServices stop/wait/session release to the worker before the safe-point completion callback so parser teardown cannot stall the game-thread ticker. The two handler-harness test IDs remain unchanged; no live runtime verification was performed.
- `#5-bounded-cancel-timeout` `IN-REVIEW` developer — Added strict timeout validation, monotonic worker deadline handling, JobRegistry cancellation, worker-only cleanup/session release, and handler-level forced-slow timeout coverage. No live runtime verification was performed.
- `#6-cooperative-abort-quiescence` `IN-REVIEW` developer — Added provider-loop abort checks, exact output ownership cleanup, pre-existing-output rejection, and a worker-quiesced test signal before hook reset or directory deletion. No live runtime verification was performed.
- `#7-output-reservation-abandonment-rollback` `IN-REVIEW` developer — Added process-local canonical output-directory reservation and rollback of abandoned-request publications; retained cooperative deadline wording. No live runtime verification was performed.
