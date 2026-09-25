---
id: B-run-tests-filter-job-never-terminal
title: "system.run_tests filter job stays 'running' forever after the automation queue drains"
status: IN-REVIEW
severity: High
category: bug
tags: [system, automation, jobs, completion-signal, run_tests, proxy-wedge]
encounters: 2
costly: 1
lastSeen: 2026-09-24T01:19:30Z
---

# system.run_tests filter job stays 'running' forever after the automation queue drains

A filter-mode, in-process `system.run_tests` job never reaches a terminal state, even when every
test finished and the engine logged its queue-drain marker. `system.job_status` keeps reporting
`status: "running"` with a new `tests running, Ns elapsed` progress event every 10 s, and
`system.job_cancel` cannot stop it (filter jobs register no cancel hook), so the ticket is stuck
for the life of the editor.

Repro (UE 5.8, host `unreal-fpv-new`, plugin `8748c637`):
1. `system.run_tests {filter: "App.Api.Device+App.UI.Login+App.Acceptance"}`.
2. PDS.log: `Automation Test Queue Empty 57 tests performed`, 57 `Test Started`, 57
   `Result={Success}` (01:19:05Z).
3. `system.job_status` on `j_20260924T011850_cc284e30`: still `running` afterwards.

It reproduces on an all-green run, so it is not caused by a failed or timed-out test.

Impact, and why it is High rather than friction: a streaming client (the bundled stdio proxy with
a progress token) blocks on the job until it is terminal. The proxy serves one request at a time
(`serve_stdio`, `mcp_proxy.py:2925`), so every later `call` from that MCP connection queues behind
the never-ending stream and hangs silently until the client's own 1800 s idle abort. See
`B-stdio-proxy-wedged-behind-endless-stream`.

**Workaround:** trust the host log (`Test Completed` lines and the `<N> tests performed` marker),
not the job status. Pass `wait: false` so the call returns the ticket instead of streaming.
**Fix (proposed):** complete the filter job from the queue-drain event the console path actually
raises (or poll the controller's test state), not only from `OnTestsComplete`, which the
`Automation RunTests` console path evidently does not fire into the handler's lambda.

## History
- `#1-stuck-after-drain` `OPEN` reporter — First job `j_20260923T210830_4e5912df` (same filter)
  drained at 21:09:12Z with 57 tests performed and was still `running` at 01:16Z (14851 s, progress
  every 10 s). An agent calling through the stdio proxy stalled ~3 h, and the next agent lost a
  30 min `call` hang before finding the cause, which cost an editor restart (costly). Reproduced on
  a fresh editor with job `j_20260924T011850_cc284e30`: 57/57 green, still `running` after drain.
- `#2-keepalive-and-poll` `IN-REVIEW` developer — Two root causes, both in
  `FRunAutomationTestsByFilterJob` (`Handlers/System/SystemControlHandler.cpp`). (1) Use-after-free:
  the `OnTestsComplete` lambda held the only `TSharedRef` to the job, and `Finish()` removes that
  binding while it is being broadcast; UE's multicast `Remove` -> `Unbind` destroys the executing
  lambda at once (`MulticastDelegateBase.h:360`, `DelegateBase.h:421`), so the job was freed mid-`Finish`:
  the completion never reached the registry, the run lease was never released
  (`AUTOMATION_RUN_IN_PROGRESS` on every later call) and `Finish` kept writing to freed heap
  (`TestsCompleteHandle.Reset()`, `MoveTemp(OnComplete)`). The exact-tests job was immune only
  because its core ticker holds a second ref and `FTSTicker::RemoveTicker` defers. Fix: the lambda
  copies `Self` to a stack local before calling. (2) A filter that matches nothing never calls
  `RunTests`, so `OnTestsComplete` never fires; the job now also polls
  `Controller->GetTestState()` from a ticker (`PinWrightRunTests::PollFilterRun`): drained once the
  controller has been Running and no longer is, or `NO_TESTS_MATCHED` if it never starts within 90 s.
  Found alongside: `FJobRegistry::AllocateId` took the GUID's leading 8 hex digits, which on Linux
  (UUIDv7) are timestamp bits, so two jobs started in the same second shared one ticket id and the
  later `Start` silently replaced the earlier ticket; now takes the random tail (`JobRegistry.cpp`).
  Baseline on HEAD code (Linux, live editor): `filter: PinWright.NoSuchTestGroupXyz` logged
  `Queue Empty 0 tests performed` and stayed `running`; a fresh editor running a 10-test filter
  (`PinWright.editor.quit+PinWright.system.run_tests.Filter`) logged `Queue Empty 10 tests performed`
  and `Sending StopTestSession` (the line immediately before `TestsCompleteDelegate.Broadcast()`),
  then stayed `running` and held the lease. After: the same zero-match filter failed
  `NO_TESTS_MATCHED` at 88 s; a 37-test filter job reached `failed/TESTS_FAILED` in the same frame as
  its drain marker (the one red is `PinWright.system.run_tests.ConcurrentJobLease`, which needs the
  process lease and cannot pass when launched from inside a `system.run_tests` job; it passes via
  console `Automation RunTests`). Tests: `PinWright.system.run_tests.CompletionLambdaKeepsJobAlive`,
  `PinWright.system.run_tests.FilterRunPollReachesTerminal`,
  `PinWright.state.job_registry.SameSecondIdsAreDistinct`. Filter jobs still register no cancel
  hook (unchanged, reported honestly by `system.job_cancel`). Plugin commit `f39443c6`.

