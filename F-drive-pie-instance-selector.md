---
id: F-drive-pie-instance-selector
title: "drive.* cannot target the listen-server PIE instance: in a host + client PIE session the game surface only exposes the client's UMG roots, so host-side UI cannot be observed or clicked"
status: OPEN
severity: Medium
category: feature
tags: [drive, drive.observe, drive.click, pie, multi-pie, listen-server, game-surface, live-root, world-selector]
encounters: 1
costly: 1
lastSeen: 2026-09-29T13:05:00Z
---

# drive.* has no PIE-instance selector

In a multi-instance PIE session (`editor.play {numClients: 2, netMode: "listen"}`: the listen server
renders in the level-editor viewport, client 1 in its own `Pioneer Drone Sim Preview [NetMode: Client 1]`
window), `drive.observe {surface: "game"}` only enumerated the **client's** UMG roots. The candidates
named by `AMBIGUOUS_LIVE_ROOT` were `JoinLeaveHUDNativeOverlay, W_OverallUILayout_C_0`, and
`W_OverallUILayout_C_0` was the client's layout: its `BecomeSpectatorButton` had
`geometry.absolute` (2454,1319), inside the client window (1880,850 646x520), the Set-of-Mark
screenshot was 640x482 and showed the client HUD (no host-only "Start" button), and the listen-server
viewport at the same moment showed the host HUD with "Start". The host's roots were not listed at all,
and `root_index` / `instance_name` select within the one PIE instance only.

`editor.console_command` and `editor.pie_status` already take a `server` / `client:N` selector;
`drive.*` has none, so any test of host-only UI (a room-admin panel, kick buttons, a host Start
button) cannot use `drive.observe` / `drive.click` at all, including the `os_input:true` path.

**Workaround:** screenshot the X display, read coordinates off the image, and inject raw
`xdotool mousedown/mouseup` at those coordinates (after checking the window under the pointer
belongs to your editor PID). Verification has to come from logs or `property.get`, since
`drive.observe` cannot see the host UI either.

**Fix (proposed):** accept the same `world` selector as `editor.console_command`
(`server` | `client` | `client:N` | `pie:N`) on every game-surface `drive.*` verb, and enumerate
live roots from that instance's `UGameViewportClient` instead of the single global game viewport.
List the instance next to each candidate in `AMBIGUOUS_LIVE_ROOT`.

## History
- `#1-host-room-panel-unreachable` `OPEN` reporter - Filed from a PDS multiplayer repro (room-admin panel on the host) on UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`, listen-server PIE with 1 client in a race lobby on `L_PDS_Stadium`. The host panel `W_MultiplayerUsersFrame` was unreachable through `drive.*`; every host click had to be done with raw xdotool from display screenshots. Costly: about 20 extra calls, and it pushed a shared-display click risk onto the operator.
