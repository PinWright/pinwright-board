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

**Files:** `Source/PinWright/Private/Handlers/System/JobControlHandler.cpp:10`.

## History
- `#1-no-job-control-rpcs` `OPEN` reporter — After a handler returns `{jobId, started:true}` there are no RPCs to poll completion, list active jobs, or cancel an in-progress operation; the JSONL stream is the only signal and requires external tooling.
- `#2-job-control-rpcs-added` `IN-REVIEW` developer — Three handlers registered via `REGISTER_RPC_HANDLER` in `JobControlHandlers.cpp`. `job_status` and `job_cancel` require `jobId` param; `job_list` takes no params. Error code `JOB_NOT_FOUND` added to the system domain.
- `#3-verified-three-rpcs-live` `DONE` tester — Verified: `system.job_list` returned full ticket array with `{ticket_id, method, status, started_at}` for every dispatched job; `system.job_status` returned `{ticket_id, method, status, started_at, completed_at, result|error, progress}` for both completed and running jobs (e.g. completed `level.save` → `result: {saved:true}`, completed `editor.screenshot` → `error: "NO_VIEWPORT"`); `system.job_cancel` returned `{cancelled:true}` on a running ticket and `{cancelled:false}` on an already-completed one. Note: param name on the RPC is `ticket_id` (not `jobId` as documented in #2 — implementation rename, not a bug).
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. Path is singular at HEAD: `JobControlHandler.cpp`, holding all three verbs — `system.job_status` `:10`, `system.job_list` `:29`, `system.job_cancel` `:49`. **Not just a path — the retention semantics the body documents are inverted.** The body says completed jobs are removed immediately and `JOB_NOT_FOUND` is the finished signal; at HEAD completed tickets are **retained** until a TTL (`JobRegistry.cpp:247-258`, `EvictExpired` drops only non-`running` tickets older than `TtlSeconds`), `system.job_list` advertises “active and recently-completed” (`:30`), and the error code is `ERR_TICKET_NOT_FOUND` (“may have been evicted”, `:21-22`). A caller following this DONE ticket would treat a completed job's real payload as an error. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
