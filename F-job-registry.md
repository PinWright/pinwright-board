---
id: F-job-registry
title: "FJobRegistry — in-memory async job tracker"
status: DONE
severity: High
category: feature
tags: [async, jobs, registry, infrastructure]
---

# FJobRegistry — in-memory async job tracker

Previously there was no shared registry for tracking the lifecycle of long-running operations. Each handler that returned early (fire-and-forget) had no way to report completion or failure back to callers.

`FJobRegistry` is a new singleton owned by `FPluginState` that assigns each async operation a UUID `jobId` at dispatch time, records the method name and creation timestamp, and allows any callback to call `Complete(jobId, bSuccess, ResultJson)` when the operation finishes. The registry is thread-safe (all mutations go through a `FCriticalSection`). `RegisterJob(Method)` returns the new `jobId`; `CompleteJob(JobId, bSuccess, Message)` fires the optional `FJobMonitorLog` write and removes the entry. `CancelJob(JobId)` is also supported for operations that can be aborted mid-flight.

**Files:** `Source/PinWright/Private/State/JobRegistry.h`, `State/JobRegistry.cpp`.

## History
- `#1-no-job-registry` `OPEN` reporter — Need an in-memory ticket registry to track long-running RPC status and serve `system.job_status` queries; no such registry exists in the plugin.
- `#2-registry-implemented` `IN-REVIEW` developer — `FJobRegistry` added to `FPluginState` alongside existing `FBlueprintTracker`. Thread-safe `TMap<FString, FJobEntry>` keyed on UUID. Completion delegates stored per-entry for `FJobMonitorLog` notification. `CancelJob` sends synthetic failure event to monitor log before removing the entry.
- `#3-verified-registry-live` `DONE` tester — Verified: kicked `editor.save_all` → returned `ticket_id: "j_20260427T024453_8c8bce49"`, `system.job_status` reported `status: "completed"` with full result payload; `system.job_list` returned all tickets with stable UUIDs (`j_<timestamp>_<hex>`); `system.job_cancel` on running `level.build_lighting` ticket returned `{cancelled: true}`. Registry tracks lifecycle correctly.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
