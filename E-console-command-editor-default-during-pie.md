---
id: E-console-command-editor-default-during-pie
title: "editor.console_command defaults to the editor world while a standalone PIE session runs, so a game command (App.Launch) reports success:true/consumed:true and does nothing"
status: OPEN
severity: Low
category: ergonomic
tags: [editor, console-command, pie, world-targeting, silent-success]
encounters: 1
lastSeen: 2026-09-25T09:26:00Z
---

# A game console command sent without `world` runs in the editor world during PIE

With one standalone PIE session running, `editor.console_command {command:"App.Launch DA_Stadium_Trainig02"}`
answered `success:true, consumed:true, world:"editor", worldPath:"/Game/System/FrontEnd/Maps/L_Core.L_Core"`.
Nothing launched; the only trace was the project log line
`LogApp: Error: No game running - start PIE or launch the game.` Resending with `world:"server"` worked.

The `world` selector (F-console-command-world-target) is documented, and `consumed` is documented as
"recognised, not succeeded", so this is not a contract break. The trap is that the default silently picks
the editor world even when exactly one PIE world exists, which is the case where a caller almost always
means the game, and the response carries no hint that a PIE world was available.

Repro: UE 5.8, host `unreal-fpv`, plugin `61c243f5`. `editor.play {}` (standalone), then
`editor.console_command {command:"App.Launch DA_Stadium_Trainig02"}`.

**Workaround:** always pass `world:"server"` (standalone PIE is its own authority) for game commands.
**Fix options:** when `world` is omitted and a single PIE world exists, add a `pieWorldAvailable` hint /
warning to the response (cheapest), or default to that PIE world for commands the editor world does not
own.

## History
- `#1-app-launch-consumed-in-editor-world` `OPEN` reporter - Found during a school-computer cross-version check (PIE against a local backend): one wasted launch and a log search to see that the command ran in the editor world.
