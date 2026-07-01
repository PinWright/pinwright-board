---
id: E-job-status-omits-test-selection
title: "system.job_status running snapshot omits requestedTests/resolvedTests"
status: WONTFIX
severity: Low
category: ergonomic
tags: [system, jobs, automation, job_status, docs]
---

# system.job_status running snapshot omits requestedTests/resolvedTests

For a `system.run_tests` job, the test-selection metadata (`requestedTests`,
`resolvedTests`, `missingTests`, `selectionMode`) is present in the
**kickoff** response and in the **terminal** result payload, but a
`system.job_status` snapshot taken while the job is still `running` does
**not** echo those fields. A caller who wants to confirm what a tracked run
actually requested/resolved — without having held onto the original kickoff
response — has no way to read it back from the live job; they must poll
`job_status` until the job reaches a terminal state and read it from the
result payload instead.

This is awkward because the selection metadata is already known at dispatch
time (it's computed before `RunTests()` and is in the kickoff payload), so a
running snapshot omitting it forces redundant polling purely to surface
data the registry could already return. The natural expectation is that
`job_status` is the authoritative read-back for a tracked job and echoes the
same selection fields regardless of running/terminal state.

**What it should do:** Include `requestedTests`/`resolvedTests`/`missingTests`
(and `selectionMode`) in the `system.job_status` snapshot for `run_tests`
jobs while they are `running`, not only at kickoff and terminal. If enriching
the payload is out of scope, document in the `system` wiki overlay
(`docs/wiki-src/system.md`) exactly which fields are available in the running
snapshot vs. the terminal result so callers know not to expect selection
metadata mid-run and don't poll-to-terminal just to read it back.

**Workaround:** Capture `requestedTests`/`resolvedTests` from the kickoff
response and keep them; or poll `system.job_status` until terminal and read
them from the result payload.

## History
- `#2-wontfix-premise-false-and-already-documented` `WONTFIX` developer — Won't fix. The ticket's central justification is false for the most-wanted fields: `resolvedTests`/`missingTests` are NOT known at dispatch and are NOT in the kickoff payload. They are computed asynchronously mid-run inside `FRunAutomationTestsByNameJob::HandleTestsRefreshed` (SystemControlHandler.cpp:260-272), after the automation controller refreshes its discovered-test list — well after the job goes `running`. At the instant a running snapshot is taken, the registry genuinely cannot return them because resolution has not happened. The only dispatch-time fields are `selectionMode`+`requestedTests`, set into `Args.StartedPayload` (SystemControlHandler.cpp:452-453) and echoed into the kickoff response by `FHandlerContext::StartJob` (HandlerContext.cpp:577-581) — i.e. already in the response the caller received; the residual read-back gap is caller-side ("I didn't keep the kickoff response"), with a stated workaround. Surfacing them in the running snapshot would require persisting kickoff context onto `FJobTicket` (JobRegistry.h:17-29) and re-emitting from `FJobRegistry::ToJson` (JobRegistry.cpp:181-204) — a deliberate lifecycle-only registry contract (F-job-registry, F-handler-context-startjob, both DONE) consumed by ~16 async handlers — disproportionate shared-infra churn for a Low-severity ergonomic nicety where every call in the originating audit succeeded. The docs-only fallback the ticket proposes is already satisfied: `docs/wiki-src/system.md:100-106` documents the running snapshot precisely (status / started_at / latest progress[] / terminal-only result|error), and the `system.run_tests` overlay (system.md:88) states selection fields land in the job payload. Behavior is real and confirmed, but it is documented-as-designed, not a defect worth the registry-redesign risk.
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean system.job_cancel task (18 calls, all ok). Friction note: "a running job_status snapshot does NOT echo requestedTests/resolvedTests — those only appear in the kickoff response and the terminal result payload, so I had to poll to terminal to confirm them." Task flow confirmed it: broad ticket a7e15f0f and narrow ticket 0ceb4d07 were both tracked, but the running snapshots (e.g. narrow ticket polled 3x while running) carried only status/started_at, not the selection metadata, which the caller wanted to confirm mid-run. Existing F-system-run-tests-multiple-names documents these fields in the kickoff/terminal payloads only; no ticket covers the running-snapshot gap. Ergonomic, not an outcome bug (every call succeeded). The separate AUTOMATION_NOT_READY terminal on the narrow run was self-reported as a real runtime result, not friction, so it is intentionally not filed here.
