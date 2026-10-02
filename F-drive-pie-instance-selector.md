---
id: F-drive-pie-instance-selector
title: "drive.* cannot target the listen-server PIE instance: in a host + client PIE session the game surface only exposes the client's UMG roots, so host-side UI cannot be observed or clicked"
status: IN-REVIEW
severity: Medium
category: feature
tags: [drive, drive.observe, drive.click, pie, multi-pie, listen-server, game-surface, live-root, world-selector]
encounters: 5
costly: 4
lastSeen: 2026-10-01T08:55:00Z
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
- `#2-capture-steals-host-clicks` `OPEN` reporter - Second encounter, same host and plugin, fix-verification pass (listen PIE, host + 1-3 clients). `drive.*` again could not reach the host's room panel. On top of that, real host clicks were silently swallowed while another PIE instance's viewport or the host's own game viewport held Slate mouse capture (`drive.input_state`: `cursor_captor SPIEViewport` / `SViewport`, `CapturePermanently`, `os_cursor` frozen at the last click point). They only landed after Shift+F1, or after the ESC menu opened and was dismissed. `drive.input_state` reports one global captor with no PIE-instance attribution, so it could not tell which instance held the capture. Costly: about 15 extra calls and several retries per session.
- `#3-screenshot-also-client-1-offscreen` `OPEN` reporter - Third encounter, UE 5.8.2 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin `2580e7f4`, listen-server PIE with 1 client (PDS race-results repro, where the host and the client tables had to be compared). The drive click and read verbs again resolved only to client 1. The **game-viewport screenshot has the same gap**: it also always captured client 1, so the host's results table could not be captured through it and was taken with an editor-window screenshot instead. The editor ran offscreen (`visible:false`), so this ticket's xdotool workaround was not available either. The fix should put the same `world` selector on the game-viewport screenshot as on `drive.*`. Not marked costly: the editor-window screenshot was enough for a read-only comparison.
- `#4-game-surface-flips-between-instances` `OPEN` reporter - Fourth encounter, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `1044f6de`, visible editor, listen PIE with 1 client (PDS race-results repro). New detail: the game surface is not even stably the client. `drive.observe {surface: game, instance_name: W_OverallUILayout}` first returned client 1's layout (geometry inside the client window), and after real clicks had gone to the host viewport the same call returned the host's `W_RaceOnlineResultsForPilotFrame_C_0` (geometry inside the level-editor viewport); both roots are named `W_OverallUILayout_C_0`, so nothing in the response says which instance answered. A handle read from one observe then failed `drive.click` with `TARGET_NOT_FOUND` on the next call because the surface had flipped back. Separately, `drive.click {os_input: true}` on the client's button returned `no_change_within_budget` with nothing clicked while the host viewport held `CapturePermanently` (as in #2); Shift+F1 via xdotool released it and the next click landed. Workaround: host clicks with raw xdotool at coordinates read from `import -window` screenshots, PID-checked with `xdotool getwindowpid`. Costly: ~20 extra calls.
- `#5-host-ui-unreachable-again` `OPEN` reporter - Fifth encounter, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `1044f6de`, visible editor, listen PIE with 1 client (PDS results-button repro). `drive.observe {surface: game}` again only resolved the client's roots, so every host-side click (СТАРТ, РЕЗУЛЬТАТЫ, ЗАКРЫТЬ) was raw xdotool against screenshot coordinates. New detail for the input side: one PIE viewport's mouse capture leaves an X pointer grab confined to whichever PIE window has focus (Xorg `XF86LogGrabInfo` showed the editor PID holding a core pointer grab, `confine 0x340006c`). XTEST clicks then reach neither window's UMG, and `drive.input_state.os_cursor` stays stale (it reported (1269,693) / (1920,443) while the X pointer was elsewhere) with no PIE-instance attribution. The workaround was to change X focus to the other PIE window, which drops the grab, plus Shift+F1 on the host. One race start could not be clicked at all and had to go through `ke B_DroneGameMode_C RestartToMapMode Play` on the server world. Costly: about 40 extra calls across several PIE sessions.
- `#6-world-selector-and-no-ambient-default` `IN-REVIEW` developer - "Added `world` (editor.console_command grammar: server | client | client:N | pie:N) to every game-surface drive verb (observe, expect, wait_for and the six action verbs via one `DRIVE_WORLD_SELECTOR_PARAM`); it rides `FDriveRootSelector::World`, so handle resolution, settle sampling and observation all use the same instance. Roots, backing map and window now come from that instance's `UGameViewportClient` (world context), never `GEngine->GameViewport`, which the editor tick leaves on whichever PIE context ticked/drew last - the root cause of #4's flip. Omitted `world`: the only PIE instance with a game viewport (a dedicated server has none), and `TARGET_AMBIGUOUS` listing every instance when several have one, built on the shared `PieWorldSelector::ResolveGameWorld` (the ui.* runtime contract, same omitted-world rule as editor.console_command) with only the has-a-game-viewport filter added; `server` on a dedicated server is `GAME_VIEWPORT_NOT_FOUND`, an unmatched selector `WORLD_NOT_FOUND`, a malformed one / 'editor' `INVALID_ARGUMENT`. `drive.observe` reports `world` (pie:N, re-passable) and `world_kind`, and its Set-of-Mark screenshot captures that instance's viewport (#3's screenshot gap; `CaptureAnnotated` gained an optional viewport-client argument). Not covered: `drive.input_state` still reports one global captor with no instance attribution (#2/#5 input-side detail), and target-less `drive.key` still types into the focused widget (world scopes only its settle observation). Files: `Handlers/Drive/DriveLiveResolver.{h,cpp}` (SelectPieInstance / ResolveGameSurface), `DriveHandlerCommon.{h,cpp}`, `DriveSetOfMarkRenderer.{h,cpp}`, `DriveTypes.h`, `DriveJson.cpp`, `DriveObserveHandler.cpp`, `DriveExpectHandler.cpp`, `DriveWaitHandler.cpp`, `DriveActionHandlers.cpp`, `docs/wiki-src/drive.md`, `CHANGELOG.md`. Tests: `PinWright.drive.game_surface.PieInstanceSelection` (pure resolver over fake standalone / listen+client / dedicated+clients instances; a live multi-instance PIE session is not started in automation) and the live single-instance `world` checks inside `PinWright.drive.game_surface.TwoUmgRootsObservedTogether`."
