---
id: B-ui-stop-play-exec-noop
title: ui.stop_play AND ui.play_in_editor always fail (STOP_FAILED / PLAY_FAILED) — unrecognized Exec strings
status: IN-REVIEW
severity: Medium
category: bug
tags: []
encounters: 1
lastSeen: 2026-07-01T15:08:20.3222269+03:00
---

## What's wrong

Both `ui.*` PIE aliases are **100% non-functional** — they drive PIE through editor
**Exec strings that no editor Exec handler consumes**, then gate success on the always-`false`
bool those Exec calls return:

- `ui.stop_play` (no params) returns `[STOP_FAILED] Failed to stop play in editor` on **every**
  valid call while PIE is active, and **leaves PIE running**.
- `ui.play_in_editor` (no params, the twin) returns `[PLAY_FAILED] Failed to start play in editor`
  and **never starts PIE**.

Each has a documented near-duplicate under `editor.*` (`editor.stop` / `editor.play`) that the
wiki says to "prefer" and that works correctly. So two documented, registered, discoverable
methods never perform their documented job.

The wiki pages advertise these as working equivalents of the `editor.*` verbs
(*"Near-duplicate of editor.stop — prefer editor.stop … retained for symmetry with the ui.*
runtime ops."* and the analogous line for `editor.play`) — but they are not.

## Root cause (verified in source + against the engine)

Both aliases call `GEditor->Exec(nullptr, <verb>)` with a string that is not a routable editor
Exec command, and gate success on that Exec's bool return:

`Plugins/PinWright/Source/PinWright/Private/Handlers/UI/UiHandler.cpp`
```cpp
// ui.stop_play
bool bCommandSuccess = GEditor->Exec(nullptr, TEXT("Stop Play In Editor"));   // -> false -> STOP_FAILED
// ui.play_in_editor
bool bCommandSuccess = GEditor->Exec(nullptr, TEXT("Play In Editor"));         // -> false -> PLAY_FAILED
```

Verified against `C:\UE_5.7\Engine\Source\Editor\UnrealEd`: there is **no** `FParse::Command`
Exec handler for `PLAY` / `STOP` / "Play In Editor" / "Stop Play In Editor` — the strings
"Play In Editor" appear only as localized display text (`NSLOCTEXT`), never as command handlers.
So both Exec calls return `false`, the `*_FAILED` branch is taken every time, and no play/stop is
ever requested. The working `editor.*` twins use the correct deferred APIs instead:

`Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/PIEHandler.cpp`
```cpp
GEditor->RequestPlaySession(PlayParams);   // editor.play  (~:73)
GEditor->RequestEndPlayMap();              // editor.stop  (:97)
```

## What it should do

Route both `ui.*` aliases through the same deferred engine APIs their `editor.*` twins use, and
return success without gating on synchronous startup/teardown (both requests are deferred to the
next tick, exactly as `editor.play` / `editor.stop` already do):

- `ui.stop_play` -> `GEditor->RequestEndPlayMap()` (mirroring `editor.stop`).
- `ui.play_in_editor` -> `GEditor->RequestPlaySession(PlayParams)` with `WorldType =
  PlayInEditor` and the level editor's first active viewport as destination when available
  (mirroring `editor.play`).

## Verbatim repro (replayed at HEAD via mcp__pinwright__call)

1. `editor.play` `{}` -> `{"success":true}`
2. `system.inspect.get_player_controllers` `{}` ->
   `{"playerControllers":[{"objectPath":".../PhysicsDemoPlayerController_C_0",...}]}` (PIE up)
3. `ui.stop_play` `{}` -> **`[STOP_FAILED] Failed to stop play in editor`**
4. `system.inspect.get_player_controllers` `{}` -> **still** present — PIE NOT stopped
5. `editor.stop` `{}` -> `{"success":true}`; then `get_player_controllers` -> `[]` (PIE actually gone)

The `ui.play_in_editor` twin was not in the original replay (step 1 used `editor.play`), but it
carries the byte-identical fake-Exec defect one method up in the same file and the same
`GEditor->Exec` shape with no routable handler, so it is equally dead and is fixed alongside
`ui.stop_play` (scope-expanded during review — see `#2`).

severity rationale: impact=blocker-with-workaround (both methods never work; the documented
siblings `editor.stop` / `editor.play` do the job and the wiki already signposts them) ×
reach=start/stop-PIE are common ops but agents are steered to the `editor.*` verbs -> Medium.

## History

- `#1-initial-repro` `OPEN` reporter — Filed: `ui.stop_play` returns STOP_FAILED on every valid call during PIE and never stops the session; root cause is the unrecognized `GEditor->Exec(nullptr, TEXT("Stop Play In Editor"))` at UiHandler.cpp:203 gating success on a bool that is always false, vs `editor.stop`'s `GEditor->RequestEndPlayMap()` at PIEHandler.cpp:97. Replay-confirmed at HEAD (PIE still up after the call; editor.stop then cleanly tore it down).
- `#2-reword-both-aliases-deferred-api` `IN-REVIEW` developer — Reworded (title/body/severity) to cover BOTH `ui.*` PIE aliases: the adversarial validity lens found the ticket under-scoped — the twin `ui.play_in_editor` (UiHandler.cpp) carries the byte-identical `GEditor->Exec(nullptr, TEXT("Play In Editor"))` fake-Exec defect (no routable engine handler; verified against C:\UE_5.7\Engine\Source\Editor\UnrealEd), gates success on an always-false bool -> PLAY_FAILED, and had no ticket of its own. Fixed both in `Plugins/PinWright/Source/PinWright/Private/Handlers/UI/UiHandler.cpp`: `ui.stop_play` now calls `GEditor->RequestEndPlayMap()` and returns success (mirroring editor.stop); `ui.play_in_editor` now calls `GEditor->RequestPlaySession(PlayParams)` with `WorldType=PlayInEditor` + the level editor's first active viewport as destination (mirroring editor.play), with an added `!GEditor` guard and the necessary IAssetViewport/LevelEditor/LevelEditorPlaySettings includes (guarded, distinct MCP_UI_* macros). Same root-cause class as the DONE ticket B-set-view-mode-exec-failed. Added regression test `PinWright.ui.pie_aliases.RouteThroughDeferredEditorApi` (Tests/UI/TestUiPieAliasesDeferredApi.cpp): headless, it asserts `ui.play_in_editor` now succeeds via the deferred RequestPlaySession (revert -> PLAY_FAILED fails it) then immediately `CancelRequestPlaySession()` so no PIE actually starts, and pins `ui.stop_play`'s no-PIE NOT_PLAYING contract (never STOP_FAILED). Not yet compiled/tested — verification pending.
