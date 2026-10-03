---
id: E-editor-not-running-cannot-distinguish-crash
title: "`EDITOR_NOT_RUNNING` reads the same for a crashed editor as for a cold one, and unconditionally prescribes `editor_start`"
status: DONE
severity: Medium
category: ergonomic
tags: [gateway, editor-not-running, error-message, crash, multi-agent, editor-start, diagnosis]
encounters: 1
lastSeen: 2026-09-02T19:47:00Z
---

# `EDITOR_NOT_RUNNING` cannot tell a crash from a cold start, and its remedy is the one an asset agent must not run

## Symptom

Every `mcp__pinwright__call` against a dead gateway returns exactly one string, whatever killed it:

```text
EDITOR_NOT_RUNNING: The Unreal editor is not running or PinWright MCP is unavailable.
Start it with the editor_start MCP tool, then retry call(). (connection refused)
```

Two problems, both hit live:

1. **The two situations it conflates need opposite responses.** A *cold* editor means "start it and
   proceed". An editor that *crashed mid-session* means "stop, the shared editor's owner has to
   decide whether to restart, and every other agent's unsaved in-memory work is already gone". The
   message gives no signal which one happened, so the caller must go outside the MCP channel —
   `Get-CimInstance Win32_Process`, `Get-NetTCPConnection` on the port in
   `Saved/PinWright/gateway-port`, and a hand-parse of `Saved/Logs/<Project>.log` for the
   `=== Critical error: ===` block — to find out.

2. **The prescribed remedy is exactly the forbidden action in a shared editor.** Multi-agent waves
   brief their asset-authoring agents with "never call `editor_start` / `editor_restart`, the lead
   owns the editor" precisely because a restart discards every other agent's unsaved state. The
   error text tells each of those agents to do it anyway. An agent that obeys its brief has to
   ignore the tool's own instruction; an agent that obeys the tool damages the wave.

## Why the gateway can do better

PinWright already holds the facts needed to separate the cases, with no new plumbing:

- `Saved/PinWright/gateway-port` exists and still names the last-serving port (27145 in the
  observed run) long after the process is gone — a stale file is itself the tell that the gateway
  once served and stopped.
- `Saved/PinWright/jobs.jsonl` carries the last completed ticket and its timestamp, so the client
  can say when the editor was last alive.
- The editor log is at a known path and its last `=== Critical error: ===` block is the crash
  reason.

## What should happen

`EDITOR_NOT_RUNNING` should distinguish at least two shapes, and drop the unconditional imperative:

```text
EDITOR_NOT_RUNNING (crashed): the editor served this project until 19:44:57Z
(last job j_...ec8acc66, method python.execute) and is gone. Crash reason:
EXCEPTION_ACCESS_VIOLATION at UUserWidget::RebuildWidget --
Saved/Logs/EAContentExamples58.log. If you share this editor with other agents,
report rather than restart; `editor_restart` discards their unsaved work.
```

```text
EDITOR_NOT_RUNNING (never started): no gateway breadcrumb for this project.
Start it with editor_start.
```

Minimum viable version: keep one message but add the two facts a caller cannot get from it today —
*whether a gateway breadcrumb exists* (crashed vs cold) and, when it does, *the path of the log to
read* — and make the `editor_start` sentence conditional on the cold case.

**Workaround:** on `EDITOR_NOT_RUNNING`, do not act on the message. Check
`Saved/PinWright/gateway-port` against the live listeners, list `UnrealEditor.exe` processes and
match their command lines to this `.uproject`, then read the last `=== Critical error: ===` block
in `Saved/Logs/<Project>.log`. Report; do not restart a shared editor.

severity rationale: impact=wasted out-of-band diagnosis on every caller, plus a live incentive to
run the one verb that destroys other agents' work x reach=every agent in every multi-agent wave,
on every editor death -> Medium. Not High: it misleads rather than corrupts, and the crash it
follows is already tracked at Critical
(`B-screenshot-designer-leaves-designer-open-compile-crash`).

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8, shared editor, port 27145) while starting Niagara explosion-emitter authoring. The first call of the task, an `asset.duplicate` of `/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst`, returned `EDITOR_NOT_RUNNING ... Start it with the editor_start MCP tool ... (connection refused)`. The brief for that agent forbids `editor_start`/`editor_restart` outright, so the message's only instruction was unusable, and establishing what had actually happened took four out-of-band probes: `Get-Process UnrealEditor*` (five engine processes, none for this project — `Get-CimInstance Win32_Process` command lines placed them on `EAContentExamples57` and `unreal-fpv/PDS`), `Get-NetTCPConnection` on the port in `Saved/PinWright/gateway-port` (nothing listening), `jobs.jsonl` (last completed job 19:43:25Z), and the log's last `=== Critical error: ===` block (19:44:57Z, `EXCEPTION_ACCESS_VIOLATION reading address 0x38` at `UUserWidget::RebuildWidget`, the crash tracked in `B-screenshot-designer-leaves-designer-open-compile-crash`). Every one of those facts is already reachable from data PinWright itself writes. Cross-ref: that ticket's `#5` entry records the same event from the authoring side.
- `#2-last-session-evidence` `IN-REVIEW` developer — `Content/Python/mcp_proxy.py`: new module functions `last_session_evidence(port_file, uproject)` and `editor_not_running_text(evidence, detail)` (plus `_last_critical_error`), and `Proxy._last_session()`; `Proxy._editor_not_running_text` is now an instance method over them and `_editor_unavailable_result` adds `structuredContent.lastSession` to every `EDITOR_NOT_RUNNING`. `state`: `never_started` (no `Saved/PinWright/gateway-port`; the only text that still says "Start it with the editor_start MCP tool"), else from the newest `Saved/Logs/<Project>*.log` written since the port file's mtime (`-CRC` crash-reporter logs excluded) `crashed` (last `=== Critical error: ===` block -> `crashReason` + `crashFrame`, Windows `LogWindows: Error:` and Unix bare-line shapes both parsed), `killed_externally` (reuses F-memory-aware-editor-launch's `_external_kill_fields` over `<log>.result.txt`), `exited` (`RequestExit(` / `Log file closed`), or `stopped`. Also `lastPort`, `gatewayBoundAt`, `lastJob` (last `jobs.jsonl` line: ts, ticket_id, method, event), `logPath`, `logStoppedAt`. Non-cold text: `EDITOR_NOT_RUNNING (<state>): an editor served this project on port N (gateway bound ...), last job ... and is not serving it now. Its log ... Crash reason: ... at <frame>. If you share this editor with other agents, report this rather than start or restart it ...`. Proxy without `--port-file`: unchanged generic text, `lastSession: null`. Limits: an `-Abslog` log outside `Saved/Logs` is not found (`logPath: null`, text says so — this is what the live wt2 checkout reads today, its editors log under a scratchpad); an editor still starting before it binds reads `stopped`. Tests: new `Content/Python/tests/test_mcp_proxy_not_running.py` `EditorNotRunningEvidenceTest` (7: never-started, Linux crash with last job, Windows critical-error block, newest log + clean exit, stale/CRC log excluded, supervisor external kill, no-port-file text). Full `Content/Python/tests` under the engine python: 452 OK, 5 skipped. Docs: `docs/wiki-src/mcp-transport.md` (new `EDITOR_NOT_RUNNING says which kind of not running` bullet), `CHANGELOG.md` (behaviour change).
- `#3-review-fixes` `IN-REVIEW` developer — Independent review follow-up, `Content/Python/mcp_proxy.py` `last_session_evidence` / `editor_not_running_text`: (1) a log counts as the serving editor's only if its `Log file open` header (local time, via the existing `_log_opened_at`, new `_read_head`) is no later than the breadcrumb mtime AND it was written since, so a commandlet / cook / failed boot / transport-less editor started after the bind is no longer blamed for a crash; (2) `<log>.result.txt` is ignored when older than the log minus 60 s, so a previous run's KILLED_EXTERNALLY no longer outranks a later clean exit; (3) `Engine exit requested` is an exit marker; (4) an undecodable port file (`UnicodeDecodeError`) reads as no breadcrumb instead of raising; (5) wording: the never-started text names the retracted-breadcrumb and still-booting cases, the docstring no longer claims the file survives every exit, and the non-cold text says the editor "did not answer this call" (true on the forward-exception path too) instead of "is not serving it now". New tests in `test_mcp_proxy_not_running.EditorNotRunningEvidenceTest`: `test_log_opened_after_the_bind_is_another_process`, `test_stale_result_file_does_not_outrank_a_later_clean_exit`, `test_undecodable_port_file_is_not_a_breadcrumb_crash` (10 total). Full `Content/Python/tests`: 461 OK, 5 skipped. `docs/wiki-src/mcp-transport.md` bullet updated to match.
- `#4-verified-linux` `DONE` tester — Fix commit 9784aa0a. Python run3: 462 tests OK (5 skipped; `test_mcp_proxy_not_running.py` has no skip sites). This includes the 10 `EditorNotRunningEvidenceTest` cases: never-started, Linux crash with last job, Windows critical-error block, newest log + clean exit, stale/CRC log excluded, supervisor external kill, no-port-file text, log opened after the bind is another process, stale result file does not outrank a later clean exit, and undecodable port file. Acceptance: `EDITOR_NOT_RUNNING` now separates `never_started` from `crashed` / `killed_externally` / `exited` / `stopped` using the gateway breadcrumb. It gives the log path, crash reason and frame, and the last job. The `editor_start` instruction appears only in the cold case, and the non-cold text says to report rather than restart a shared editor. `structuredContent.lastSession` is machine-readable. Coverage limits: the Windows log shape is parsed from fixture text, not a live Windows crash. An `-Abslog` log outside `Saved/Logs` is not found (the text says so). An editor still booting before it binds reads `stopped`.
