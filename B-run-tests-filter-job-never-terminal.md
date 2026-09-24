---
id: B-run-tests-filter-job-never-terminal
title: "system.run_tests filter job stays 'running' forever after the automation queue drains"
status: OPEN
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
