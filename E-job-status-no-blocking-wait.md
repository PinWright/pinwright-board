---
id: E-job-status-no-blocking-wait
title: "system.job_status has no bounded wait or multi-ticket form, so waiting on parallel wait:false jobs means a jobs.jsonl polling loop"
status: OPEN
severity: Low
category: ergonomic
tags: [system, jobs, job_status, async, wait, polling, asset, dump-folder]
encounters: 1
lastSeen: 2026-09-30T00:00:00Z
rice: [2, 1, 1, 2]
priority: 8
---

# system.job_status has no bounded wait or multi-ticket form, so waiting on parallel wait:false jobs means a jobs.jsonl polling loop

`system.job_status` declares a single required `ticket_id` and answers with the current snapshot immediately (`Handlers/System/JobControlHandler.cpp:10-27`). There is no `waitSeconds`/timeout parameter and no way to pass several tickets. The only blocking path is SSE block-and-stream on the *kickoff* call (`HandlerContext.cpp:709-712`, `RegisterStreamingJob`), which is per-request: an agent that fans out N jobs with `wait:false` to run them concurrently, or any plain-JSON client (always gets a ticket), must then poll. The wiki offers only the poll or `jobs.jsonl` (`docs/wiki-src/system.md:26-30`, `:287-291`).

The plumbing a bounded wait needs already exists: `FJobRegistry::OnJobEvent()` (`State/JobRegistry.h:71`, `:114`) is the multicast the subsystem's streaming bridge subscribes to (`PinWrightSubsystem.cpp:366`) to resolve a deferred HTTP request at a terminal event (`PinWrightSubsystem.cpp:698`).

**Workaround:** `wait:false` kickoffs, then either call `system.job_status` per ticket repeatedly or grep `Saved/PinWright/jobs.jsonl` for `"event":"completed"|"failed"|"cancelled"` per ticket (match `event`, not `status`, per `system.md:57`).

**Fix:** add optional `waitSeconds` to `system.job_status` (clamped below the HTTP request timeout, `HttpDefaultTimeoutMs`/`HttpMaxTimeoutMs` in `Public/PinWrightSettings.h:61`, `:65`): if the ticket is already terminal reply at once; otherwise defer the response, resolving on the first terminal `OnJobEvent` for that ticket or on timeout with the `running` snapshot plus `timedOut: true`. Must not block the game thread (defer via the existing completion path, not a sleep). Optionally accept `ticket_ids: [...]` returning per-ticket snapshots, with the wait resolving when all (or, with `mode: "any"`, the first) are terminal. Document on `### system.job_status`.

## History
- `#1-parallel-dump-poll-script` `OPEN` reporter — A subagent kicked off 10 `asset.dump_folder` jobs with `wait:false` to run them concurrently, then had to write `waitjob.sh`, which greps `jobs.jsonl` every 5 s per ticket, because `system.job_status` accepts only `ticket_id` and returns immediately. Verified from source at plugin `2580e7f4`: no wait/timeout or multi-ticket parameter on `system.job_status` or `system.job_list`; the only blocking mechanism is the kickoff-time SSE stream. Friction only; every job completed.
