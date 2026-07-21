---
id: F-console-command-world-target
title: "editor.console_command cannot target a specific PIE world (server/client)"
status: IN-REVIEW
severity: Medium
category: feature
tags: [editor, console-command, pie, multiplayer, world-targeting]
encounters: 1
lastSeen: 2026-07-21T00:00:00Z
---

# editor.console_command cannot target a specific PIE world (server/client)

`editor.console_command` only executes in the editor world and
`system.console_command` runs at process scope — no RPC can direct a console
command at a specific PIE world. MP-in-PIE testing needs exactly that: running
`App.Launch ... servertravel` in the PIE **server** world and
`open 127.0.0.1:17777` in the PIE **client** world. Origin: MP sumo netcode
test sessions 2026-07-21, where every such command was forced through a
`python.execute` workaround (`unreal.SystemLibrary.execute_console_command`
with hand-rolled world iteration) that runs synchronously on the game thread
and stalls the editor when it polls.

**Workaround:** `python.execute` + `unreal.SystemLibrary.execute_console_command`
with manual iteration over world contexts to find the right PIE world; synchronous
on the game thread, stalls the editor under polling.
**Fix:** Optional `world` param on `editor.console_command` —
`"editor"` (default) | `"server"` | `"client[:N]"` | `"pie:N"` — resolved via
`GEngine` world contexts (`EWorldType::PIE`, `PIEInstance`, NetMode
classification). On no match, return a helpful error listing the available
contexts.

## History
- `#1-mp-pie-targeting-gap` `OPEN` reporter — Filed from MP sumo netcode test sessions: server/client-scoped console commands (App.Launch servertravel in the PIE server world, `open` in the PIE client world) have no RPC path and fall back to a game-thread-stalling python.execute world-iteration workaround. Implementation of the `world` param is in flight by another agent.
- `#2-implemented-verified` `IN-REVIEW` developer — Implemented in PinWright commit 73a229d1: `world` selector ("editor" default | "server" | "client[:N]" 1-based | "pie:N") resolved via a pure PieWorldSelector helper; INVALID_ARGUMENT on malformed selectors, WORLD_NOT_FOUND listing available contexts on a miss. 4 new handler tests green; live-verified in a 2-client listen session (selector drove App.Launch servertravel on the server context; both error paths exercised verbatim). Awaiting independent tester verification for DONE.
