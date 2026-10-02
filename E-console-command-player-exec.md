---
id: E-console-command-player-exec
title: "editor.console_command cannot run PlayerController / CheatManager exec commands (EnableCheats, summon) in a PIE world; returns EXEC_FAILED"
status: DONE
severity: Medium
category: ergonomic
tags: [editor, console-command, pie, cheat-manager, player-controller, exec]
encounters: 2
lastSeen: 2026-09-29T16:25:00Z
---

# Player-routed exec commands are unreachable through editor.console_command

In listen PIE on `L_PDS_Stadium`:

```
editor.console_command {command:"EnableCheats", world:"server"}
-> EXEC_FAILED: No exec command or console variable consumed 'EnableCheats'.
editor.console_command {command:"summon /Script/App.ReplayPlayerState", world:"server"}
-> EXEC_FAILED (same)
```

Both are standard commands typed in the game console. They are `exec` functions on
`APlayerController` / `UCheatManager`, which the in-game console reaches through the local player
(`ULocalPlayer::Exec` -> `PlayerController->ConsoleCommand`). `editor.console_command` apparently goes
through `GEngine->Exec(World, ...)` only, so player-owned exec handlers never see the command. The
error text ("check the spelling ... unloaded module") points the wrong way.

**Workaround:** `python.execute` with
`unreal.SystemLibrary.execute_console_command(world, cmd, unreal.GameplayStatics.get_player_controller(world, 0))`,
which routes through the player controller. `EnableCheats` then `summon` both worked this way.

**Fix (proposed):** when a PIE world is targeted and `GEngine->Exec` does not consume the command,
retry through that world's first local `APlayerController::ConsoleCommand`. At minimum, mention
player/cheat exec commands in the EXEC_FAILED message and on the wiki page.

## History
- `#1-enablecheats-summon` `OPEN` reporter - Filed from the PDS QA #744 PlayerIndex repro, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`. Needed `summon` on the server world; worked around with `python.execute` in two extra calls.
- `#2-viewmode-too` `OPEN` reporter - Second encounter, same host/plugin: `editor.console_command {command:"viewmode lit", world:"server"}` returned the same EXEC_FAILED (`viewmode` is a UGameViewportClient exec). The PIE game viewport had been switched to wireframe by a stray F1 key. Same `python.execute` + `SystemLibrary.execute_console_command(world, cmd, pc)` workaround. Cheap.
- `#3-re-rated` `OPEN` triage — Severity Low -> Medium. This is not pure friction: the verb cannot run PlayerController/CheatManager/GameViewportClient exec commands at all, its EXEC_FAILED text points at spelling or unloaded modules, and the only route is an undocumented `python.execute` + `SystemLibrary.execute_console_command(world, cmd, pc)` workaround. That is the Medium soft-blocker band; no reach modifier applies.
- `#4-local-player-exec-fallback` `IN-REVIEW` developer - In a PIE world (explicit or the new sole-PIE default), a line `GEditor->Exec` does not consume is retried through the world's first local player, `ULocalPlayer::Exec` (the in-game console route: game viewport client, game instance, then `UPlayer::Exec` over PlayerInput, PlayerController, pawn, HUD, GameMode, CheatManager, GameState, camera manager). So `EnableCheats`, `summon` and `viewmode` now run. The response reports `route: "engine"|"localPlayer"` and `playerController` on the player route. `EXEC_FAILED` text now says why: editor world (player commands only exist in a PIE world), PIE world with no local player (dedicated server: target a client), or the player chain was tried too. Files: `Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp`, `docs/wiki-src/editor.md`. Test: `PinWright.editor.console_command.PiePlayerExecOmittedWorld` clears the PIE PlayerController's `CheatManager`, sends `EnableCheats`, and requires success, `route:"localPlayer"`, the receiving controller's name, and a recreated `CheatManager` (read-back); also asserts explicit `world:"editor"` still refuses `EXEC_FAILED` pointing at a PIE world. Fails on revert (EXEC_FAILED). Not exercised: dedicated-server branch and `summon`/`viewmode` (same chain as `EnableCheats`).
- `#5-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.editor.console_command.PiePlayerExecOmittedWorld` passed in w23-final: with the PIE PlayerController's CheatManager cleared, `EnableCheats` succeeds with `route:"localPlayer"`, names the receiving controller, and the CheatManager is recreated (read back); explicit `world:"editor"` still refuses `EXEC_FAILED` pointing at a PIE world. The proposed fix (retry through the world's local player) and the clearer EXEC_FAILED text are met. Limits: `summon` and `viewmode` were not sent (same `ULocalPlayer::Exec` chain); the dedicated-server branch was not exercised.
