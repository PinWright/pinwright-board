---
id: B-console-command-world-precondition-ensure
title: "`editor.console_command` during PIE trips the WorldPrecondition ensure (\"Handler-defined world 'pie:0' disagrees with resolved target\") on every success response"
status: OPEN
severity: Medium
category: bug
tags: [editor, console-command, world-precondition, ensure, pie, crash-reporter]
encounters: 1
lastSeen: 2026-09-23T18:49:49Z
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
