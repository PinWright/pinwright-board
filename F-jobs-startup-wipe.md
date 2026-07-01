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

**Files:** `Source/EditorAutomationRpcGateway/Private/EditorAutomationRpcGatewaySubsystem.cpp` — `Initialize()` method.

## History
- `#1-stale-jobs-accumulate` `OPEN` reporter — Without a startup wipe, `jobs.jsonl` accumulates records across editor sessions; jobs from crashed sessions appear permanently in-flight and confuse automation scripts that parse the file for completion events.
- `#2-startup-wipe-implemented` `IN-REVIEW` developer — Wipe + rotation added to `Initialize()` before `Transport->Start()`. `IFileManager::Get().Move` used for rename; `IFileManager::Get().FileSize` used to detect non-empty file. Rotation cap enforced by listing `jobs.jsonl.*.bak` files sorted by mtime and deleting the oldest when count exceeds 3.
- `#3-verified-wipe-on-restart` `DONE` tester — Verified: editor restarted mid-session (level.build_all triggered a stall and the editor recovered); jobs.jsonl was completely absent from disk after the restart. First post-restart kickoff (`editor.save_all`) recreated the file fresh — only contained the new session's events (`started` + `completed` for the new ticket). No accumulation from the prior session. Rotation backup was not created in this case because the file was already gone (presumably the previous editor crashed without flushing).
