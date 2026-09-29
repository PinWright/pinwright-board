---
id: B-console-command-world-precondition-ensure
title: "`editor.console_command` during PIE trips the WorldPrecondition ensure (\"Handler-defined world 'pie:0' disagrees with resolved target\") on every success response"
status: OPEN
severity: Medium
category: bug
tags: [editor, console-command, world-precondition, ensure, pie, crash-reporter]
encounters: 6
lastSeen: 2026-09-29T13:05:00Z
---

# A successful console command raises an engine ensure while decorating its response

With PIE running on `/Game/System/FrontEnd/Maps/L_Core`, both of these produced a handled ensure and a crash
report folder (`Saved/Crashes/UECC-...`), with a ~2.4 s hitch while the report was written:

- `editor.console_command {command: "DisableAllScreenMessages", world: "pie:0"}`
- `editor.console_command {command: "DisableAllScreenMessages"}` (default `world`, i.e. editor)

```
Ensure condition failed: ExistingWorld.Equals(ResolvedWorld, ESearchCase::IgnoreCase)
  [WorldPrecondition.cpp:148]
Handler-defined world 'pie:0' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'
  PinWrightWorldPrecondition::AddWorldField            WorldPrecondition.cpp:148
  UPinWrightSubsystem::DecorateAutomationResponse      PinWrightSubsystem.cpp:625
  FHandlerContext::SendSuccess                         HandlerContext.cpp:440
  AutoHandler_328_                                     EditorCommandHandler.cpp:392
```

The handler writes its own `world` field (the selector string, e.g. `pie:0`, or a PIE world path), then
`AddWorldField` compares it to the dispatcher's resolved editor world and ensures on the mismatch. The
two are different vocabularies (selector vs. object path; PIE world vs. editor world), so the ensure fires
on a normal success. The response itself is fine.

`B-editor-dies-silently-cef-pie-lobby` already noted four of these ("world 'server'") and explicitly filed
no ticket; this is that ticket.

**Impact:** noise in `Saved/Crashes`, a multi-second hitch per call, and a real risk that a genuine crash
is mistaken for this ensure (it was, here: the same session had a fatal crash minutes later).

**Fix (proposed):** in `AddWorldField`, skip the equality ensure when the handler-set world is a PIE
selector or a PIE world, or have `editor.console_command` report its target under a different key
(`targetWorld`) and leave `world` to the dispatcher.

## History
- `#1-ensure-on-console-command-in-pie` `OPEN` reporter - Filed from a UMG pass on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Two `editor.console_command` calls during PIE (one with `world: "pie:0"`, one default) each raised the ensure above and wrote a crash report (`UECC-Windows-F5BEB123...`, `UECC-Windows-B666C3AC..._0000`).
- `#2-worked-around-via-python` `OPEN` reporter - UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Avoided `editor.console_command` during PIE (`App.ListTracks`, `App.ListExperiences`, `App.Launch ...`, `App.CompleteRace`) and ran them with Python `unreal.SystemLibrary.execute_console_command(<PIE world>, cmd)`; no ensure. Output then has to be read from `Saved/Logs/PDS.log`. Cheap.
- `#3-fires-outside-pie-editor-and-server` `OPEN` reporter - Third sighting, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`. Two handled ensures today, both from `editor.console_command {command: "App.Launch mapeditor"}` (not from editor startup, as first suspected: the stack is `AutoHandler_322_` `EditorCommandHandler.cpp:392` -> `FHandlerContext::SendSuccess` `HandlerContext.cpp:440` -> `DecorateAutomationResponse` `PinWrightSubsystem.cpp:625` -> `AddWorldField` `WorldPrecondition.cpp:148`; the ensure line has moved to `:150`). `Saved/Logs/PDS-backup-2026.09.28-08.51.29.log:4031`: `Handler-defined world 'editor' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`, with no PIE running (the command itself logged `No game running`), so the default selector trips it too, not only PIE selectors. `Saved/Logs/PDS.log:3940`: `Handler-defined world 'server' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`. Cause unchanged: `EditorCommandHandler.cpp:382` writes the selector (`editor` / the caller's `world`) into `world` and `AddWorldField` (`WorldPrecondition.cpp:137-156`) compares it case-insensitively with the resolved world path, which can never match. Cheap: ~30-45 ms `FDebug::EnsureFailed` each, no work lost.
- `#4-server-selector-wt1-listwaves` `OPEN` reporter - Fourth sighting, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux), plugin `61c243f5`. First `editor.console_command {command: "au.Debug.ListWaves", world: "server"}` of a standalone PIE on `L_Core` raised the handled ensure (`Handler-defined world 'server' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`, stack through `AutoHandler_322_` -> `SendSuccess` -> `DecorateAutomationResponse` -> `AddWorldField`). Response itself was correct. Cheap: ~30 ms, fires once per session.
- `#5-three-editor-starts-app-launch` `OPEN` reporter - Fifth sighting, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`. `editor.console_command {command: "App.Launch mapeditor", world: "server"}` in standalone PIE on `L_Core` produced the handled ensure (`Handler-defined world 'server' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`) in each of three editor sessions (PIDs 3049871, 3147326, 3256675). Each left an `ensureinfo-PDS-pid-<pid>-*` folder under `Saved/Crashes`. The command itself worked every time. Cheap.
- `#6-listen-pie-app-launch-wt1` `OPEN` reporter - Sixth sighting, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux), plugin `61c243f5`, listen-server PIE with 2 instances (`editor.play {numClients:2, netMode:"listen"}`) on `L_Core`. `editor.console_command {command: "App.Launch race draft:autosave loc=DA_Stadium online=lan backend=lan servertravel room=AdminRepro", world: "server"}` returned success and raised the handled ensure (`Handler-defined world 'server' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`, `WorldPrecondition.cpp:150`, stack via `AutoHandler_322_` `EditorCommandHandler.cpp:392`). Cheap: the command itself ran; the ensure only cost a crash-report write and log noise.
