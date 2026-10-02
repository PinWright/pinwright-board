---
id: E-console-command-editor-default-during-pie
title: "editor.console_command defaults to the editor world while a standalone PIE session runs, so a game command (App.Launch) reports success:true/consumed:true and does nothing"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [editor, console-command, pie, world-targeting, silent-success]
encounters: 2
lastSeen: 2026-09-30T12:26:00Z
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
- `#2-re-rated` `OPEN` triage — Severity Low -> Medium. The default path returns `success:true, consumed:true` for a game command that did nothing while a PIE world is running, which is close to the High silent-false-success class on a normal path; held one level lower because the response echoes `world:"editor"` and `consumed` is documented as recognised-not-succeeded.
- `#3-second-app-launch-editor-world` `OPEN` reporter - Second sighting, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `adb239fd`, standalone PIE running on `L_Core`. `editor.console_command {command:"App.Launch freeflight"}` answered `success:true, consumed:true, world:"editor", worldPath:"/Game/System/FrontEnd/Maps/L_Core.L_Core"`; the log only had `LogApp: Error: No game running - start PIE or launch the game.` Resent with `world:"server"` and it launched. Cost: one launch and a log grep.
- `#4-omitted-world-follows-sole-pie` `IN-REVIEW` developer - `editor.console_command` with `world` omitted now resolves by what is running: no PIE world -> editor world (unchanged), exactly one PIE world -> that PIE world, several PIE worlds -> refused `TARGET_AMBIGUOUS` with the PIE contexts listed (never a pick). Explicit `world:"editor"` still means the editor world during PIE. The response echoes the resolved selector in `world` (`pie:N` when defaulted), plus `worldDefaulted` and `route`; the `EXEC_FAILED` message now names the world it tried. `world` changed from `RPC_PARAM_DEF(..."editor")` to `RPC_PARAM_OPT`. Files: `Source/PinWright/Private/Handlers/Editor/PieWorldSelector.h` (`EOmittedWorld` + `ResolveOmitted`), `Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp`, `docs/wiki-src/editor.md`. Tests: `PinWright.editor.console_command.OmittedWorldResolve` (pure 0/1/several rule) and `PinWright.editor.console_command.PiePlayerExecOmittedWorld` (owned host-neutral standalone PIE; `EnableCheats` with `world` omitted must answer from the PIE world's path; fails on revert because the editor world does not consume it), in `Source/PinWright/Private/Tests/EditorOps/TestConsoleCommandPieRouting.cpp`. The PIE test inherits the host PIE-start log-error caveat (`LogApp` `FRelayClient` on the PDS host).
