---
id: E-console-command-player-exec
title: "editor.console_command cannot run PlayerController / CheatManager exec commands (EnableCheats, summon) in a PIE world; returns EXEC_FAILED"
status: OPEN
severity: Low
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
