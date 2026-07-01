---
id: F-job-control-rpcs
title: "system.job_status / job_list / job_cancel — job control RPCs"
status: DONE
severity: High
category: feature
tags: [async, jobs, rpc, system, monitoring]
---

# system.job_status / job_list / job_cancel — job control RPCs

After a fire-and-forget handler returns `{jobId, started:true}`, callers need a way to poll for completion, list all active jobs, and cancel an in-progress operation. Without these RPCs the JSONL stream is the only signal, and it can only be read by external tooling.

Three new RPC methods were added in `Handlers/System/JobControlHandlers.cpp`:

- **`system.job_status`** — takes `{jobId}`, returns `{jobId, method, status, startedAt}` or `{error: JOB_NOT_FOUND}` if the job has already completed and been removed.
- **`system.job_list`** — returns `{jobs: [{jobId, method, startedAt}, ...]}` for all currently active (in-flight) jobs.
- **`system.job_cancel`** — takes `{jobId}`, calls `FJobRegistry::CancelJob`, writes a synthetic cancel event to `jobs.jsonl`, returns `{jobId, cancelled:true}` or `JOB_NOT_FOUND`.

Completed jobs are removed from the registry immediately on completion, so `job_status` returning `JOB_NOT_FOUND` is the signal that the job finished (callers should read `jobs.jsonl` for the outcome).

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/System/JobControlHandlers.cpp`.

## History
- `#1-no-job-control-rpcs` `OPEN` reporter — After a handler returns `{jobId, started:true}` there are no RPCs to poll completion, list active jobs, or cancel an in-progress operation; the JSONL stream is the only signal and requires external tooling.
- `#2-job-control-rpcs-added` `IN-REVIEW` developer — Three handlers registered via `REGISTER_RPC_HANDLER` in `JobControlHandlers.cpp`. `job_status` and `job_cancel` require `jobId` param; `job_list` takes no params. Error code `JOB_NOT_FOUND` added to the system domain.
- `#3-verified-three-rpcs-live` `DONE` tester — Verified: `system.job_list` returned full ticket array with `{ticket_id, method, status, started_at}` for every dispatched job; `system.job_status` returned `{ticket_id, method, status, started_at, completed_at, result|error, progress}` for both completed and running jobs (e.g. completed `level.save` → `result: {saved:true}`, completed `editor.screenshot` → `error: "NO_VIEWPORT"`); `system.job_cancel` returned `{cancelled:true}` on a running ticket and `{cancelled:false}` on an already-completed one. Note: param name on the RPC is `ticket_id` (not `jobId` as documented in #2 — implementation rename, not a bug).
