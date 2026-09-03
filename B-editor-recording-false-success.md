---
id: B-editor-recording-false-success
title: "`editor.start_recording` reports a replay started even when edit mode has no GameInstance and the DemoRec command does nothing"
status: OPEN
severity: High
category: bug
tags: [editor, replay, recording, world-selection, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Demo recording succeeds on the wire without starting a recorder

## What happens

`editor.start_recording` selects the PIE world when present, otherwise the editor
world, executes `DemoRec`, ignores the `Exec` result, and unconditionally returns
`success:true` with `Recording started` (`EditorCommandHandler.cpp:932-960`). It
also returns success if no world was resolved.

UE 5.8's `UWorld::HandleDemoRecordCommand` starts a replay only when the world has a
`GameInstance`, but returns true even when that condition is false
(`C:\UE_5.8\Engine\Source\Runtime\Engine\Private\World.cpp:5760-5782`). An
editor world has no gameplay instance, so the handler's documented edit-mode path
is a consumed no-op even if its ignored `Exec` return were checked.

## Why it matters

Callers can finish a capture workflow believing a `.demo` is being recorded when no
recorder or artifact exists. Severity is High for silent false success on the
verb's primary contract.

## What should happen

Require a game/PIE world with a valid `GameInstance`, start through the replay API,
and verify the replay driver/recording state or wait for the relevant completion
signal before success. Return the resolved world and actual replay name/path. In
edit mode, return `NO_ACTIVE_GAME_WORLD` instead of success.

## Workaround

Start PIE first and independently verify replay recording state or the output file;
do not treat this verb's current response as evidence.

## Related

- `B-createpackage-unvalidated-paths-plugin-wide` — separately tracks unsafe name
  composition, not whether replay recording starts.

## History
- `#1-source-scan-demorec-noop` `OPEN` reporter — Source-only scan followed the edit-world fallback into UE 5.8's DemoRec handler, where a null `GameInstance` skips `StartRecordingReplay` but still consumes the command; PinWright always reports success. No build, test, editor, MCP call, or plugin edit was performed.
