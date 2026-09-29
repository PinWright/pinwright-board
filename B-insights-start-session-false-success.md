---
id: B-insights-start-session-false-success
title: "insights.start_session reported status:\"started\" whatever Trace.Start did; stop_session returned before the file closed; snapshot failure returned success"
status: IN-REVIEW
severity: High
category: bug
tags: [insights, trace, false-success, report-only-what-happened, gap-analysis-2026-09-28]
encounters: 1
costly: 0
lastSeen: 2026-09-29T14:00:00Z
---

# insights.start_session false success

`insights.start_session` ran the `Trace.Start` console command and then answered
`{status: "started"}` unconditionally. When a trace was already open, the engine refused
(`LogCore: Error: Unable to start trace, already tracing to <dest>`) and nothing started, but the
verb still reported success. The same thing happened when the call came right after a stop:
`FTraceAuxiliary::Stop` only queues the close, and TraceLog's worker thread completes it shortly
after. Until then `IsConnected()` stays true, so a start in that window was refused. The response
never named the file it wrote, so a caller could not find the trace it thought it had started.

Same pattern in the sibling verbs:
- `insights.stop_session` returned while the close was still queued. The reported `.utrace` could
  still be open, and a following `start_session` or `export_trace` ran into the pending close.
- `insights.snapshot` answered success with `status: "snapshot_failed"` when
  `FTraceAuxiliary::WriteSnapshot` returned false.
- `insights.set_channels` echoes the requested channel lists as `enabled` / `disabled` and reports
  nothing measured.

Evidence: `Saved/Logs/pw_gapwave_full_offscreen2.log`, where test-side starts in
`InsightsHandlerTest.cpp` hit both engine errors (`Unable to start trace, already tracing to` and
`Trace failed to connect`). Engine source: `TraceAuxiliary.cpp` `TraceAuxiliaryStartShared` (the
Error), TraceLog `Writer.cpp` `Writer_Stop` / `Writer_IsTracing` (pending close keeps
`IsTracing` true).

## History
- `#1-unconditional-started` `OPEN` reporter — Found while fixing the insights tests exposed by `B-suppress-log-errors-static-leaks`: `start_session` hardcodes `status: "started"` after `GEngine->Exec("Trace.Start")`, so an already-open or still-closing trace is reported as a new session that never started. `stop_session` returns before the close completes, and `snapshot` reports failure as success.
- `#2-measured-start-stop-snapshot` `IN-REVIEW` developer — `Handlers/Debug/InsightsHandler.cpp`:
  - **start_session** calls `FTraceAuxiliary::Start(File, nullptr, channels)` directly. A pending close (connected, connection type `None`) is waited out for up to 5 s through the new `PinWrightRpc::Insights::WaitForTraceClose`.
  - A trace that is still open is refused with `TRACE_ALREADY_ACTIVE`. The payload carries `activeDestination`, `connectionType` and `connected`, and the open trace is not touched.
  - When the engine opens no connection the verb returns `TRACE_START_FAILED`. On success it reports the measured `tracePath`, `connected`, `connectionType` and `activeChannels`.
  - **stop_session** waits up to 5 s for the close and reports `closed`. `status` is `stopped` / `stop_pending` (with a warning) / `not_running`.
  - **snapshot** failure is `SNAPSHOT_FAILED`.
  - **set_channels** keeps its request echo and adds the measured `activeChannels`.
  - New codes in `Handlers/ErrorCodes.h`; `WaitForTraceClose` is declared in `Handlers/Debug/InsightsHandlerInternal.h`.
  - Tests in `Tests/InsightsHandlerTest.cpp`: `insights.start_session.RefusesWhileTraceActive` checks the refusal code, that the payload names the active destination, and that the open trace is unchanged. `insights.start_session.WaitsOutPendingClose` issues `start_session` inside a real pending close, asserts it succeeds with the engine's live destination, then asserts `stop_session` reports `stopped` / `closed` for the same path.
  - Wiki: `docs/wiki-src/insights.md` method sections and the `insights.tracing-reference.md` command table.
  - Only a `-SingleFile` compile has been run; the tests have not been run. There is no test for `TRACE_START_FAILED` or `SNAPSHOT_FAILED`, because neither engine failure can be provoked deterministically.
- `#3-stop-refused-while-connecting` `IN-REVIEW` developer — Full offscreen suite `Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log` (~37001): `insights.start_session.WaitsOutPendingClose` failed with `TRACE_ALREADY_ACTIVE`, and the game thread blocked for the full 5 s wait.
  - **Root cause.** The engine does not need the game thread to close a trace; the Stop never took effect. `FTraceAuxiliary::Start` only queues the connection (TraceLog `Writer_SetPendingHandle`), and the editor's trace worker thread adopts it on its next update, every ~17 ms. Until then `Writer_Stop` returns false because `GPendingDataHandle` is set, and `FTraceAuxiliaryImpl::Stop` returns before resetting the target. The test's bare Stop came microseconds after Start, so the engine refused it and the trace simply went live.
  - Given that, the handler's `TRACE_ALREADY_ACTIVE` was the correct answer, and the test premise was wrong. The 5 s block came from the test's own scope-exit wait, whose Stop was refused the same way.
  - Every test-side stop in this file had the same problem. Each "wait for close" timed out at 5 s, and cleanup could not delete the still-open `.utrace` (`Error deleting file ... Error Code 32`).
  - **Real product defect found along the way:** `stop_session` right after `start_session` made one refused Stop and answered `not_running` while the trace kept recording.
  - **Fix.** New `PinWrightRpc::Insights::RequestTraceStop` (`Handlers/Debug/InsightsHandler.cpp`, declared in `InsightsHandlerInternal.h`). It retries `FTraceAuxiliary::Stop` for up to 5 s while a connection is pending. It returns true when the stop is accepted or an earlier stop's close is already pending (connection type `None`), and false when nothing is connected or the stop never took. `stop_session` uses it.
  - `WaitForTraceClose` now also calls `UE::Trace::Update()` each iteration. That is a no-op while the worker thread runs (the editor default) and drives the writer under `-notracethreading`. An accepted close then completes in about two worker updates.
  - **Tests.** The test helper `StopAndWaitForClose` now uses `RequestTraceStop`, as does `NotConnectedReturnsEmpty`.
  - `WaitsOutPendingClose` now enters a real pending close: `RequestTraceStop`, then the precondition connected plus type `None`, with a skip marker otherwise.
  - New `insights.stop_session.StopsATraceStillConnecting` calls `stop_session` right after Start and expects `stopped`, `closed:true`, the same path and no connection. It can only discriminate when the call lands before the worker adopts the connection; the test body runs well under the 17 ms update interval, so it normally does.
  - Wiki `insights.md` `stop_session` section updated. `-SingleFile` compile of both .cpp files succeeded. Not yet run.
