---
id: B-inspect-singletons-pick-client-world
title: "system.inspect.get_game_state / get_player_states pick the PIE client world in a listen-server session, with no world selector and no field naming which instance answered"
status: OPEN
severity: Medium
category: bug
tags: [system-inspect, get-game-state, get-player-states, pie, multi-pie, listen-server, world-selector]
encounters: 1
lastSeen: 2026-09-29T13:05:00Z
---

# PIE-first singleton accessors answer from the client in a multi-instance session

In a listen-server PIE session with one client (`editor.play {numClients: 2, netMode: "listen"}`,
both worlds on `L_PDS_Stadium`), `system.inspect.get_game_state {}` returned
`/Game/Maps/UEDPIE_1_L_PDS_Stadium.L_PDS_Stadium:PersistentLevel.B_DroneGameState_C_0` and
`system.inspect.get_player_states {}` returned the two `DronePlayerState_*` of `UEDPIE_1`, i.e. the
**client's** replicated copies. Server-authoritative state (GameMode decisions, non-replicated or
not-yet-replicated fields) is on `UEDPIE_0`. Neither method takes a `world` parameter and the
response has no `world` / `pieInstance` / `netMode` field, so nothing tells the caller it is
reading a client proxy; the wiki pages only say "active (PIE-first) world".

**Workaround:** rewrite the returned path's `UEDPIE_1_` to `UEDPIE_0_` and pass it to
`system.inspect.inspect_object` / `property.get`.

**Fix (proposed):** accept the same `world` selector as `editor.console_command`
(`server` | `client` | `client:N` | `pie:N`), default to the server/authority instance in a
networked session, and echo `world` + `netMode` in the response.

## History
- `#1-listen-pie-returns-client` `OPEN` reporter - Filed from a PDS multiplayer repro on UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`. Wanted the server `ADroneGameState::bAdminOnlyPilotActivation`; got the client world's GameState path and had to hand-edit it to the `UEDPIE_0` path. Cheap once noticed, but only noticed by reading the path prefix.
