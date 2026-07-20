---
id: B-no-editor-identity-handshake
title: "No editor-identity handshake: RPCs silently execute on a co-located same-project editor"
status: OPEN
severity: High
category: bug
tags: [transport, http, multi-editor, misroute, identity]
encounters: 1
lastSeen: 2026-07-20T00:00:00Z
---
# No editor-identity handshake: RPCs silently execute on a co-located same-project editor

PinWright derives the MCP port per project path (auto-derive on: `19880 + hash(projectPath) % 10240` → 24281 here). Two editors of the **same** project derive the **same** port, so the second fails to bind. The refuse-to-serve fix from `B-port-conflict-false-active` makes that second editor log an error and not serve (`SocketHttpServer.cpp:221-233`; caller `PinWrightSubsystem.cpp:159-178` sets `bStreamingActive=false`, logs "Transport failed to bind port", and continues — the editor orphans as a live non-serving process). But the **first** editor keeps serving on the port, and the MCP client connects to it.

The gap: no RPC returns editor identity. The `system.*` namespace exposes job/inspect/console/live-coding/`run_*` handlers but nothing carrying pid, project path, launch args, or a caller-supplied token (searched `Handlers/System`, `Handlers/Environment`). So a caller that "launched its own editor" cannot tell from any response that its RPCs are being served by a **different** (e.g. the user's interactive) editor. A mutating RPC then hits an unintended live editor.

This is distinct from `B-port-conflict-false-active`: that ticket fixes the *second* (failing) editor's self-honesty; auto-derive does **not** help here (same project path → same port), and nothing lets a *client* verify which live editor answers it.

Session evidence: an agent launched a 2nd `UnrealEditor.exe` for this project while the user's editor served 24281. Log: `HTTP transport binding port 24281.` → `LogSocketHttp: Error: Port 24281 could not be bound ...` → `Transport failed to bind port 24281.` The 2nd editor orphaned (non-serving). The client's `blueprint.set_default` (`EyeMaterialTeam0` on `/App/HELIOS/Drones/Atlas/B_PioneerSumo`) ran on the FIRST editor, which reinstanced its live drones and crashed. No response field flagged the wrong target.

**Workaround:** Before any mutating call, confirm `call()` reaches YOUR editor (attribute a read RPC's result to the instance you launched, or confirm the target editor's log shows the bind you expect). If the per-project port is already served, STOP — do not launch a second same-project editor and drive it; it will not serve, and your RPCs hit the first.
**Fix:** Add an editor-identity RPC (e.g. `system.identity`) returning pid / project path / launch args (or a caller-supplied launch token) so a client can verify its target before the first mutating call; and/or have the failed-bind editor exit (or hard-disable MCP-dependent automation) rather than orphan as a live non-serving process.

## History
- `#1-initial-repro` `OPEN` reporter — Two same-project editors share a derived port (24281); the second refuses to serve (B-port-conflict-false-active fix), but no editor-identity RPC lets the client detect its RPCs are served by the FIRST editor. A misrouted `blueprint.set_default` reinstanced the user's live drones and crashed that editor; nothing in the response indicated the wrong target.
