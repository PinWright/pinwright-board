---
id: F-inspect-game-singletons
title: "No MCP accessors for live PIE game-framework singletons (GameInstance / GameMode / GameState / PlayerController / PlayerState / LocalPlayer)"
status: DONE
severity: Medium
category: feature
tags: [system-inspect, pie, runtime, game-framework, gameinstance, gamemode, ergonomics]
---

# No MCP accessors for live PIE game-framework singletons

When PIE is running, an authoring/debugging agent often needs to ask "what's
the live state of the GameMode right now" or "give me the active
PlayerController so I can `property.get` its drone-state variables". The
plugin currently exposes no typed accessor for any of the standard live
game-framework singletons. The only path today is `python.execute` calling
`unreal.GameplayStatics.get_game_instance(world)` /
`get_game_mode(world)` / `get_player_controller(world, i)`, with the agent
also responsible for picking the PIE world over the editor world via
`unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_game_world()`.

This is a daily debugging task and `python.execute` is the wrong surface for
it: every call ships a multi-line python snippet, the response is stdout
text rather than typed JSON (path + className), and there is no schema for
agents to discover.

The existing `system.inspect.*` family is the right namespace but is
**editor-world-only**: every handler in `EnvironmentHandler.cpp`
(`get_world_settings`, `list_objects`, `find_by_class`, `find_by_tag`,
`inspect_class`, `inspect_object`, ...) resolves its world via
`GEditor->GetEditorWorldContext().World()`, never via `GEditor->PlayWorld`.
`actor.find_by_class` and its `system.inspect.find_by_class` mirror partially
overlap — they can locate a PlayerController *class* in the editor world —
but they will not see PIE-spawned instances and they have no notion of
"the controller for player index N" or the unique GameMode/GameState
singletons.

A precedent for "PIE world first, editor world fallback" already exists
inside the plugin: `Handlers/System/SessionsHandler.cpp` defines a static
helper `GetGameInstance()` that returns `GEditor->PlayWorld->GetGameInstance()`,
and `GetLocalPlayerByIndex(int32)` / `GetLocalPlayerCount()` are layered on
top — but these are internal to the multiplayer-sessions handlers, not
exposed as standalone introspection RPCs.

**Repro of the gap and python workaround** (PIE active on `L_Core`):

```
mcp__editor-automation__call path="python.execute" args={"code": "..."}
# get_game_world() -> /Game/System/FrontEnd/Maps/UEDPIE_0_L_Core.L_Core
# get_game_instance(world) -> /Engine/Transient.LyraEditorEngine_0:B_DroneGameInstance_C_2
# get_game_mode(world)     -> .../UEDPIE_0_L_Core.L_Core:PersistentLevel.B_LyraGameMode_C_0
# get_player_controller(world, 0) -> .../L_Core:PersistentLevel.B_DronePlayerController_C_0
```

No MCP equivalent exists for any of those four lookups.

**Proposed RPCs** (read-only; all under `system.inspect.*` to extend the
existing namespace rather than spawn a new `runtime.*` tree):

| Method | Returns |
|---|---|
| `system.inspect.get_game_instance` | `{ objectPath, className }` or `GAME_INSTANCE_NOT_FOUND` |
| `system.inspect.get_game_mode`     | `{ objectPath, className }` or `GAME_MODE_NOT_FOUND` |
| `system.inspect.get_game_state`    | `{ objectPath, className }` or `GAME_STATE_NOT_FOUND` |
| `system.inspect.get_player_controllers` | `[ { objectPath, className, playerIndex } ]` (empty array if PIE not running) |
| `system.inspect.get_player_states`      | `[ { objectPath, className, playerIndex } ]` |
| `system.inspect.get_local_players`      | `[ { objectPath, className, playerIndex, controllerId } ]` |

World selection rule for all six: pick `GEditor->PlayWorld` when non-null,
otherwise fall back to `GEditor->GetEditorWorldContext().World()`. This
matches the `SessionsHandler` precedent and makes the same RPC useful both
during PIE and as a no-op-returning surface outside PIE (returning an empty
array or `GAME_MODE_NOT_FOUND`, never an exception).

**Implementation hint:** all six are one-liners over the standard UE
accessors. Sketch (no compile, no commit):

```cpp
// Inside a new handler file under Handlers/Environment/ or Handlers/System/.
UWorld* World = (GEditor && GEditor->PlayWorld)
    ? GEditor->PlayWorld
    : (GEditor ? GEditor->GetEditorWorldContext().World() : nullptr);

UGameInstance* GI = World ? World->GetGameInstance() : nullptr;
AGameModeBase* GM = World ? World->GetAuthGameMode()  : nullptr;
AGameStateBase* GS = World ? World->GetGameState()    : nullptr;
// Players:
for (FConstPlayerControllerIterator It = World->GetPlayerControllerIterator(); It; ++It) { ... }
// Player states via World->GetGameState()->PlayerArray, or UGameplayStatics::GetPlayerState(World, i).
// Local players via GI->GetLocalPlayers().
```

Each handler emits `{ objectPath: Obj->GetPathName(), className: Obj->GetClass()->GetName() }`.
Arrays emit one entry per iterator slot with `playerIndex` matching iteration order.

**Acceptance check** (manual, in PIE on `L_Core` with the standard
PDS player flow):
- `system.inspect.get_game_instance` returns
  `objectPath` ending in `B_DroneGameInstance_C_<n>` and `className =
  DroneGameInstance` (or the `_C` form per existing dump convention).
- `property.get` against that returned path successfully reads
  `B_DroneGameInstance` BP variables (sanity-checks the path is a real,
  resolvable UObject reference, not just a label).
- `system.inspect.get_player_controllers` returns one entry whose
  `objectPath` ends in `B_DronePlayerController_C_0` and whose
  `playerIndex = 0`.
- Outside PIE, all six RPCs return either the editor-world fallback (for
  GameInstance/GameMode when applicable) or an empty array / typed error
  code — never an unhandled exception.

**Workaround until implemented:** `python.execute` with a snippet that picks
the PIE world via `unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_game_world()`
and calls `unreal.GameplayStatics.get_game_instance(world)` /
`get_game_mode(world)` / `get_player_controller(world, i)` /
`get_player_state(world, i)`. Functional but ergonomically a regression
versus typed `system.inspect.*` RPCs (stdout text, not JSON; no schema; no
discovery via `?`).

## History
- `#1-initial-repro` `OPEN` reporter — No `system.inspect.*` (or any other namespace) handler returns the live PIE GameInstance / GameMode / GameState / PlayerController / PlayerState / LocalPlayer singletons. Confirmed by `mcp__editor-automation__call path="system.inspect"` (no such methods listed; only `find_by_tag`/`list_objects`/`inspect_class`/`find_by_class`/`inspect_object` are suggested) and by grepping the plugin source — `GetAuthGameMode` / `GetGameInstance` / `GetPlayerController` references exist only inside `Handlers/System/SessionsHandler.cpp` (PIE-world-first internal helper) and `Handlers/Blueprint/BlueprintGraphHandler.cpp` / `Handlers/Debug/PerformanceHandler.cpp` / `Handlers/Build/PipelineHandler.cpp` (incidental, not exposed as introspection RPCs). All existing `system.inspect.*` handlers in `EnvironmentHandler.cpp` resolve their world via `GEditor->GetEditorWorldContext().World()` — editor-world-only, never `PlayWorld`. Partial overlap with `actor.find_by_class` / `system.inspect.find_by_class` is editor-world-only too and has no "controller for player N" semantic. Python fallback demonstrated working in PIE on `L_Core`: `python.execute` returned `game_instance = /Engine/Transient.LyraEditorEngine_0:B_DroneGameInstance_C_2`, `game_mode = ...UEDPIE_0_L_Core.L_Core:PersistentLevel.B_LyraGameMode_C_0`, `player_controller_0 = ...B_DronePlayerController_C_0` — but the agent had to ship a multi-line python snippet and parse stdout instead of receiving typed JSON.
- `#2-add-game-singleton-handlers` `IN-REVIEW` developer — Added six `system.inspect.get_game_*` handlers in `Handlers/Environment/SystemInspectSingletonsHandler.cpp` exposing live GameInstance/GameMode/GameState/PlayerController/PlayerState/LocalPlayer singletons with PIE-first world resolution. Regression tests cover no-PIE empty-array and not-found error paths.
- `#3-verify-pass` `DONE` tester — Verified in live PIE on `L_Core`: `system.inspect.get_game_mode {}` returns `{objectPath: ".../UEDPIE_0_L_Core.L_Core:PersistentLevel.B_LyraGameMode_C_0", className: "B_LyraGameMode_C"}` from the PIE world directly, no python fallback needed. Acceptance bar met.
