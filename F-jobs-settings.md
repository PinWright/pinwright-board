---
id: F-jobs-settings
title: "Settings additions for job registry and JSONL monitor"
status: DONE
severity: Low
category: feature
tags: [settings, async, jobs, configuration]
---

# Settings additions for job registry and JSONL monitor

The JSONL monitor path and job retention behaviour needed to be configurable through the existing `UEditorAutomationRpcGatewaySettings` UI rather than hardcoded.

New fields added to `UEditorAutomationRpcGatewaySettings`:

- **`JobsLogPath`** (`FString`, default `".editor-automation/jobs.jsonl"`) — path relative to project root where job events are written. Allows teams to redirect the file outside the project directory if needed.
- **`bEnableJobMonitorLog`** (`bool`, default `true`) — toggle the JSONL writer entirely; useful for CI environments where filesystem writes are restricted.
- **`MaxConcurrentJobs`** (`int32`, default `64`) — registry rejects new registrations above this limit with `JOB_LIMIT_EXCEEDED` to prevent unbounded growth during runaway automation scripts.

**Files:** `Source/PinWright/Public/PinWrightSettings.h`, `Private/EditorAutomationRpcGatewaySettings.cpp`.

## History
- `#1-job-settings-missing` `OPEN` reporter — JSONL monitor path and job retention behaviour are hardcoded; they need to be configurable via `UEditorAutomationRpcGatewaySettings` so teams can redirect the log path and disable filesystem writes in CI.
- `#2-settings-fields-added` `IN-REVIEW` developer — Three new `UPROPERTY` fields added under a `"Job Monitoring"` category group in the settings class. `FJobMonitorLog` reads `JobsLogPath` and `bEnableJobMonitorLog` at construction time from `GetDefault<UEditorAutomationRpcGatewaySettings>()`. `FJobRegistry` reads `MaxConcurrentJobs` on each `RegisterJob` call.
- `#3-verified-default-path-honored` `DONE` tester — Verified indirectly: every kickoff response in this session reported `monitor_path: ".editor-automation/jobs.jsonl"` and the JSONL file actually exists at that path on disk, confirming `JobsLogPath` default is read and applied. `bEnableJobMonitorLog` is implicitly `true` (file is being written). `MaxConcurrentJobs` default of 64 was not stressed in this session — not tested live, but the registry accepted ~12 concurrent tickets without `JOB_LIMIT_EXCEEDED` errors.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. Path(s) here that move by more than the prefix in this ticket, taken from the plugin's rename history rather than the prefix rule: `Source/EditorAutomationRpcGateway/Public/EditorAutomationRpcGatewaySettings.h` → `Source/PinWright/Public/PinWrightSettings.h`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
