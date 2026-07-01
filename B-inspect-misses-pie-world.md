---
id: B-inspect-misses-pie-world
title: "`system.inspect.*` and `actor.*` queries miss the active PIE world entirely"
status: DONE
severity: High
category: bug
tags: []
---

# `system.inspect.*` and `actor.*` queries miss the active PIE world entirely

`system.inspect.list_objects`, `system.inspect.find_by_class`, and
`system.inspect.find_by_tag` (and their mutating `actor.*` twins
`actor.list`, `actor.find_by_class`, `actor.find_by_tag`) only enumerate
actors in the editor `PersistentLevel`. They skip the active PIE world
entirely, so during a live PIE session you cannot locate the
`PlayerController`, `GameMode`, `GameState`, pawn, or any
runtime-spawned actor through MCP. This blocks the entire "inspect a
live PIE actor" workflow.

**Repro (PIE running on `L_Core`):**

1. Confirm PIE is live via `python.execute`:
   ```python
   import unreal
   ues = unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem)
   gw = ues.get_game_world()
   # → /Game/System/FrontEnd/Maps/UEDPIE_0_L_Core.L_Core
   pcs = unreal.GameplayStatics.get_all_actors_of_class(gw, unreal.PlayerController)
   # → 1 controller: .../UEDPIE_0_L_Core.L_Core:PersistentLevel.B_DronePlayerController_C_0
   ```
2. `system.inspect.find_by_class { "className": "PlayerController" }`
   → `{ "objects": [], "count": 0, "success": true }`
3. `system.inspect.find_by_class { "className": "GameMode" }`
   → `{ "objects": [], "count": 0, "success": true }`
4. `system.inspect.list_objects {}` → 36 actors, **all** with paths
   under the editor world
   `/Game/System/FrontEnd/Maps/L_Core.L_Core:PersistentLevel.*`. The
   PIE world `UEDPIE_0_L_Core` is absent.

**Root cause:** `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp`
hardcodes the editor world via `GEditor->GetEditorWorldContext().World()`:
- `list_objects` at lines 1362–1374
- `find_by_class` at lines 1404–1420
- `find_by_tag` at lines 1450–1465

The `actor.*` twins do the same — `actor.find_by_class` at
`Handlers/Actor/QueryHandler.cpp:287` uses `GEditor->GetEditorWorldContext().World()`,
and `actor.list` / `actor.find_by_tag` route through
`UEditorActorSubsystem::GetAllLevelActors()`, which also returns the
editor level. None of these consult `GEditor->GetWorldContexts()` for a
`EWorldType::PIE` entry.

**Workaround:** Use `python.execute` with
`unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_game_world()`
and `unreal.GameplayStatics.get_all_actors_of_class(gw, ...)` directly.

**Fix:** Add an optional `world` parameter to the six handlers:
`"editor" | "pie" | "auto"`, default `"auto"`. In `"auto"` mode, pick
the PIE world when one exists (walk `GEditor->GetWorldContexts()` for
the first context with `WorldType == EWorldType::PIE` and a non-null
`World()`), otherwise fall back to the editor world. Echo the resolved
world type and path in the response so callers can disambiguate.
Extract a shared `ResolveQueryWorld(FHandlerContext&)` helper (e.g. in
`Utils/ActorUtils.h`) since six handlers across two files need the
same logic.

## Acceptance

With PIE running on `L_Core`,
`system.inspect.find_by_class { "className": "PlayerController" }`
returns at least one entry whose `path` starts with
`/Game/System/FrontEnd/Maps/UEDPIE_0_L_Core.L_Core:PersistentLevel.`
(e.g. `B_DronePlayerController_C_0`). Same for `GameMode`.
`system.inspect.list_objects {}` includes runtime-spawned PIE actors.
With no PIE session active, behavior is unchanged (returns editor
world actors).

## History
- `#1-initial-repro` `OPEN` reporter — Reproduced live with PIE running on `L_Core`: PIE world resolves to `/Game/System/FrontEnd/Maps/UEDPIE_0_L_Core.L_Core` with 1 `B_DronePlayerController_C_0` and a `B_LyraGameMode_C_0` (confirmed via `python.execute` + `unreal.GameplayStatics`), but `system.inspect.find_by_class` returns 0 for both `PlayerController` and `GameMode`, and `system.inspect.list_objects` returns 36 actors all from the editor `L_Core` PersistentLevel. Handlers in `EnvironmentHandler.cpp:1362,1404,1450` and `QueryHandler.cpp:287` use `GEditor->GetEditorWorldContext().World()` and never check `GEditor->GetWorldContexts()` for `EWorldType::PIE`.
- `#2-add-resolve-query-world` `IN-REVIEW` developer — Added `McpActorUtils::ResolveQueryWorld` (PIE-first via `GEditor->PlayWorld`, fallback to editor world); wired optional `world` param into six handlers in `EnvironmentHandler.cpp` and `QueryHandler.cpp`; `actor.list`/`actor.find_by_tag` switched off `UEditorActorSubsystem::GetAllLevelActors()` to `TActorIterator`; responses echo resolved `world`+`worldPath`. Regression test `FResolveQueryWorldEditorModeReturnsEditorWorld` covers the helper.
- `#3-verify-pass` `DONE` tester — Verified in live PIE on `L_Core`: `system.inspect.find_by_class { className: "PlayerController" }` returns `B_DronePlayerController_C_0` under `/Game/System/FrontEnd/Maps/UEDPIE_0_L_Core.L_Core:PersistentLevel.*` with new `"world": "auto"` + `worldPath` echo fields; `find_by_class { "GameMode" }` returns `B_LyraGameMode_C_0` the same way. Acceptance bar met.
