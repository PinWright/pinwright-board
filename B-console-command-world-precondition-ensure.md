---
id: B-console-command-world-precondition-ensure
title: "`editor.console_command` during PIE trips the WorldPrecondition ensure (\"Handler-defined world 'pie:0' disagrees with resolved target\") on every success response"
status: IN-REVIEW
severity: Medium
category: bug
tags: [editor, console-command, world-precondition, ensure, pie, crash-reporter]
encounters: 9
lastSeen: 2026-09-30T08:56:58Z
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

**Root cause (not PIE-specific):** `AddWorldField` (`Dispatch/WorldPrecondition.cpp`, called from
`UPinWrightSubsystem::DecorateAutomationResponse` for every mutating response) ensured that a
handler-set `world` equals `GetWorldIdForMethod(Method)`, the editor-world object path. Handlers
document `world` in other vocabularies: `editor.console_command` echoes its selector (`editor`,
`pie:0`, ...; `docs/wiki-src/editor.md` "echoes `world` and `worldPath`"), `level.add_sublevel`
writes `World->GetName()` (short name), and the query/environment/ui handlers write a mode or short
name. So the first mutating success per session whose handler sets `world` trips it, PIE or not
(the default `world: "editor"` also mismatches). The invariant was never held, so the ensure is
wrong, not the handlers; renaming the handler fields would change documented wire contracts.

**Fix (implemented):** removed the comparison; a handler-set `world` still wins, unchanged on the
wire.

## History
- `#1-ensure-on-console-command-in-pie` `OPEN` reporter - Filed from a UMG pass on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Two `editor.console_command` calls during PIE (one with `world: "pie:0"`, one default) each raised the ensure above and wrote a crash report (`UECC-Windows-F5BEB123...`, `UECC-Windows-B666C3AC..._0000`).
- `#2-worked-around-via-python` `OPEN` reporter - UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Avoided `editor.console_command` during PIE (`App.ListTracks`, `App.ListExperiences`, `App.Launch ...`, `App.CompleteRace`) and ran them with Python `unreal.SystemLibrary.execute_console_command(<PIE world>, cmd)`; no ensure. Output then has to be read from `Saved/Logs/PDS.log`. Cheap.
- `#3-fires-outside-pie-editor-and-server` `OPEN` reporter - Third sighting, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`. Two handled ensures today, both from `editor.console_command {command: "App.Launch mapeditor"}` (not from editor startup, as first suspected: the stack is `AutoHandler_322_` `EditorCommandHandler.cpp:392` -> `FHandlerContext::SendSuccess` `HandlerContext.cpp:440` -> `DecorateAutomationResponse` `PinWrightSubsystem.cpp:625` -> `AddWorldField` `WorldPrecondition.cpp:148`; the ensure line has moved to `:150`). `Saved/Logs/PDS-backup-2026.09.28-08.51.29.log:4031`: `Handler-defined world 'editor' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`, with no PIE running (the command itself logged `No game running`), so the default selector trips it too, not only PIE selectors. `Saved/Logs/PDS.log:3940`: `Handler-defined world 'server' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`. Cause unchanged: `EditorCommandHandler.cpp:382` writes the selector (`editor` / the caller's `world`) into `world` and `AddWorldField` (`WorldPrecondition.cpp:137-156`) compares it case-insensitively with the resolved world path, which can never match. Cheap: ~30-45 ms `FDebug::EnsureFailed` each, no work lost.
- `#4-server-selector-wt1-listwaves` `OPEN` reporter - Fourth sighting, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux), plugin `61c243f5`. First `editor.console_command {command: "au.Debug.ListWaves", world: "server"}` of a standalone PIE on `L_Core` raised the handled ensure (`Handler-defined world 'server' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`, stack through `AutoHandler_322_` -> `SendSuccess` -> `DecorateAutomationResponse` -> `AddWorldField`). Response itself was correct. Cheap: ~30 ms, fires once per session.
- `#5-three-editor-starts-app-launch` `OPEN` reporter - Fifth sighting, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`. `editor.console_command {command: "App.Launch mapeditor", world: "server"}` in standalone PIE on `L_Core` produced the handled ensure (`Handler-defined world 'server' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`) in each of three editor sessions (PIDs 3049871, 3147326, 3256675). Each left an `ensureinfo-PDS-pid-<pid>-*` folder under `Saved/Crashes`. The command itself worked every time. Cheap.
- `#6-listen-pie-app-launch-wt1` `OPEN` reporter - Sixth sighting, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux), plugin `61c243f5`, listen-server PIE with 2 instances (`editor.play {numClients:2, netMode:"listen"}`) on `L_Core`. `editor.console_command {command: "App.Launch race draft:autosave loc=DA_Stadium online=lan backend=lan servertravel room=AdminRepro", world: "server"}` returned success and raised the handled ensure (`Handler-defined world 'server' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`, `WorldPrecondition.cpp:150`, stack via `AutoHandler_322_` `EditorCommandHandler.cpp:392`). Cheap: the command itself ran; the ensure only cost a crash-report write and log noise.
- `#7-fix-pass-repeats` `OPEN` reporter - Seventh sighting, same host/plugin: during the #744 fix verification the handled ensure fired twice in each of two editor sessions (`grep -c "disagrees with resolved target"` = 2 in `PDS-backup-2026.09.29-14.04.42.log` and in `PDS.log`), all from `editor.console_command` with `world: "server"` / `"pie:N"` during listen PIE. Cheap.
- `#8-app-launch-pie0-crash-report` `OPEN` reporter — UE 5.8, host `X:\src\unreal\unreal-fpv`, crash report `Saved/Crashes/UECC-Windows-FB4DFF4748BFCCC95FEA65B577BB8FBC_0000` (09:20:51 UTC, ensure-only, `PDS.log` inside). Dispatched RPC: `editor.console_command` (`id=39be2d66-...`, run inline by SafePoint) with `world: "pie:0"`, line `App.Launch DA_Arena_Sumo`; ensure text "Handler-defined world 'pie:0' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'"; stack `AddWorldField` WorldPrecondition.cpp:148 <- `DecorateAutomationResponse` PinWrightSubsystem.cpp:648 <- `SendAutomationResponse` :659 <- `FHandlerContext::SendSuccess` HandlerContext.cpp:440 <- `AutoHandler_328_` EditorCommandHandler.cpp:402 <- `FRpcDispatcher::ProcessRequest` <- `UPinWrightSubsystem::Tick`. Root-cause analysis added to the body: the ensure asserts an invariant several handlers' documented `world` fields contradict (also reachable via `level.add_sublevel` without PIE).
- `#9-removed-unheld-world-ensure` `IN-REVIEW` developer — Fix committed in plugin `29e9d445` (pinwright-ue master). Changed `AddWorldField` in `Source/PinWright/Private/Dispatch/WorldPrecondition.cpp` to return early when the handler already set `world`, dropping the `ensureMsgf` comparison against `GetWorldIdForMethod`. Response bytes are unchanged (the handler value already won); only the ensure, crash-report folder and hitch go away. Verify: in PIE, `editor.console_command {command: "stat unit", world: "pie:0"}` and default-world, then `level.add_sublevel`; no `Handled ensure` in the log and no new `Saved/Crashes/UECC-*`.
- `#10-app-launch-pie0-then-crash` `OPEN` reporter - Sighting on a plugin build that predates the #9 fix (see plugin hash), UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`. `editor.console_command {command: "App.Launch DA_ChemicalMap_Race_00 mapmode=EditMap", world: "pie:0"}` on standalone PIE `L_Core` raised the ensure (`WorldPrecondition.cpp:150`, `ensureinfo-PDS-pid-3794727-01A0EEB3D89275A0AFF60361A0F055DB`). 25 s later the editor crashed in the recorder drain (`B-recorder-drain-writeevent-fname-segv`); unrelated stack, filed separately.
- `#11-stale-binary-wt1-listen-pie` `IN-REVIEW` reporter - Sighting on a stale plugin binary, not a regression of #9. UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux). The checkout's PinWright source is `2580e7f4`, which contains `29e9d445` (`AddWorldField` returns early), but `Plugins/PinWright/Binaries/Linux/libUnrealEditor-PinWright.so` was built 2026-09-29 18:30Z, before `29e9d445` (19:53Z). In a 2-instance listen PIE on `L_Core`, `editor.console_command {command: "App.Launch race draft:autosave online=lan backend=lan servertravel room=GapRepro", world: "server"}` raised the handled ensure at `WorldPrecondition.cpp:150` (`Handler-defined world 'server' disagrees with resolved target '/Game/System/FrontEnd/Maps/L_Core.L_Core'`, stack via `AutoHandler_322_` `EditorCommandHandler.cpp:392`). The line number matches the pre-fix code. Cheap. For the tester: #9 has to be verified on a rebuilt plugin binary, not on this checkout's current binary.
