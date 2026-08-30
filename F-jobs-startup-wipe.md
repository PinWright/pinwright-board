---
id: F-jobs-startup-wipe
title: "Subsystem Initialize wipes jobs.jsonl and rotates stale logs"
status: DONE
severity: Low
category: feature
tags: [async, jobs, startup, cleanup, jsonl]
---

# Subsystem Initialize wipes jobs.jsonl and rotates stale logs

Without a startup wipe, `jobs.jsonl` accumulates records from every previous editor session. Jobs from sessions that crashed mid-operation would appear as permanently in-flight, confusing automation scripts that parse the file to detect completion.

`UEditorAutomationRpcGatewaySubsystem::Initialize` now wipes `jobs.jsonl` at startup (if `bEnableJobMonitorLog` is true in settings) before the transport starts accepting requests. If a non-empty `jobs.jsonl` already exists from a prior session, it is renamed to `jobs.jsonl.<timestamp>.bak` before the fresh file is created, keeping up to 3 rotation files (oldest deleted when the limit is exceeded). This preserves the prior session's log for post-mortem inspection without polluting the current session's view.

**Files:** `Source/PinWright/Private/PinWrightSubsystem.cpp` — `Initialize()` method.

## History
- `#1-stale-jobs-accumulate` `OPEN` reporter — Without a startup wipe, `jobs.jsonl` accumulates records across editor sessions; jobs from crashed sessions appear permanently in-flight and confuse automation scripts that parse the file for completion events.
- `#2-startup-wipe-implemented` `IN-REVIEW` developer — Wipe + rotation added to `Initialize()` before `Transport->Start()`. `IFileManager::Get().Move` used for rename; `IFileManager::Get().FileSize` used to detect non-empty file. Rotation cap enforced by listing `jobs.jsonl.*.bak` files sorted by mtime and deleting the oldest when count exceeds 3.
- `#3-verified-wipe-on-restart` `DONE` tester — Verified: editor restarted mid-session (level.build_all triggered a stall and the editor recovered); jobs.jsonl was completely absent from disk after the restart. First post-restart kickoff (`editor.save_all`) recreated the file fresh — only contained the new session's events (`started` + `completed` for the new ticket). No accumulation from the prior session. Rotation backup was not created in this case because the file was already gone (presumably the previous editor crashed without flushing).
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. Path(s) here that move by more than the prefix in this ticket, taken from the plugin's rename history rather than the prefix rule: `Source/EditorAutomationRpcGateway/Private/EditorAutomationRpcGatewaySubsystem.cpp` → `Source/PinWright/Private/PinWrightSubsystem.cpp`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
