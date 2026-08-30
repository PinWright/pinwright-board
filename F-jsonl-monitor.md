---
id: F-jsonl-monitor
title: "FJobMonitorLog — JSONL event stream at .editor-automation/jobs.jsonl"
status: DONE
severity: High
category: feature
tags: [async, jobs, jsonl, monitoring, filesystem]
---

# FJobMonitorLog — JSONL event stream at .editor-automation/jobs.jsonl

Callers needed a persistent, human-readable record of job outcomes that survives across MCP tool calls. Without it there is no way to `tail -f` or grep completion events from outside the process.

`FJobMonitorLog` is a new writer class that appends one JSON line per job lifecycle event to `<ProjectRoot>/.editor-automation/jobs.jsonl`. Each line contains `{ts, jobId, method, event, success, message}`. The writer is called by `FJobRegistry` on `CompleteJob` and `CancelJob`. Writes are synchronous but cheap (one `fwrite` + flush per event); the file is opened in append mode so multiple editor sessions accumulate without truncation — the subsystem startup wipe (see `F-jobs-startup-wipe`) handles clearing on each fresh launch.

**Files:** `Source/PinWright/Private/Utils/JobMonitorLog.h`, `State/JobMonitorLog.cpp`.

## History
- `#1-no-persistent-job-log` `OPEN` reporter — Job outcomes are only visible in-memory; callers need a persistent, human-readable JSONL stream at a known path to `tail -f` or grep completion events from outside the editor process.
- `#2-jsonl-writer-implemented` `IN-REVIEW` developer — `FJobMonitorLog` writes `<ProjectRoot>/.editor-automation/jobs.jsonl`. Registered in `FJobRegistry`; called on every `CompleteJob` / `CancelJob`. Startup wipe delegated to `F-jobs-startup-wipe` path in `EditorAutomationRpcGatewaySubsystem::Initialize`.
- `#3-verified-jsonl-events` `DONE` tester — Verified: `C:/Unity/unreal-fpv/.editor-automation/jobs.jsonl` exists at documented path. Each event line is JSON with `{ts, ticket_id, method, event, ...}` fields. `event: "started"` / `"progress"` / `"completed"` / `"failed"` all observed in the live stream during this session (asset.dump_folder progress at 1Hz, editor.save_all started+completed, editor.screenshot started+failed with NO_VIEWPORT). Append-mode writes confirmed across multiple jobs.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. Path(s) here that move by more than the prefix in this ticket, taken from the plugin's rename history rather than the prefix rule: `Source/EditorAutomationRpcGateway/Private/State/JobMonitorLog.h` → `Source/PinWright/Private/Utils/JobMonitorLog.h`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
