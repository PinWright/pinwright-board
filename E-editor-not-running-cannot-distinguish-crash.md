---
id: E-editor-not-running-cannot-distinguish-crash
title: "`EDITOR_NOT_RUNNING` reads the same for a crashed editor as for a cold one, and unconditionally prescribes `editor_start`"
status: OPEN
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
