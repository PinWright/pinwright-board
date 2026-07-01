---
id: F-inspect-list-subsystems
title: "No MCP accessor for live UEngine/UEditor/UGameInstance/UWorld/ULocalPlayer subsystem instances"
status: DONE
severity: Medium
category: feature
tags: [system-inspect, subsystem, pie, runtime, game-framework, gameinstance, editor, ergonomics]
---

# No MCP accessor for live subsystem instances

UE's subsystem framework (`UEngineSubsystem`, `UEditorSubsystem`,
`UGameInstanceSubsystem`, `UWorldSubsystem`, `ULocalPlayerSubsystem`) is the
canonical home for service-style singletons. PDS alone defines dozens
(`UApiSubsystem`, `UUsersSubsystem`, `UTrackCacheSubsystem`, …) and so does
the engine (`UAssetEditorSubsystem`, `UEditorActorSubsystem`, …). An
authoring/debugging agent that wants to read state out of one of them needs
the **live instance's object path**, but the plugin currently exposes no
typed accessor for any subsystem flavor.

The existing `system.inspect.*` family is the right namespace but is
**actor-centric** — every method (`list_objects`, `find_by_class`,
`find_by_tag`, `inspect_class`, `inspect_object`) operates over UWorld
actors. Subsystems live on the outer `UEngine` / `UGameInstance` / `UWorld`
/ `ULocalPlayer` and are not in any actor iteration. Confirmed via
`mcp__editor-automation__call path="system.inspect"` — the response lists
only the five actor-centric leaves; no `list_subsystems` or
`find_subsystem`. Grep against `Plugins/EditorAutomationRpcGateway/Source/`
for `GetSubsystemArray` / `GetEngineSubsystem` confirms no handler in the
plugin registers a subsystem-enumeration RPC.

The only working path today is `python.execute` with
`unreal.find_object(gi, 'ApiSubsystem_0')` — ergonomically bad (multi-line
python snippet, stdout text instead of typed JSON, no schema discovery)
and brittle: the agent must already know the instance suffix
(`_0`, `_1`, …) and the outer's path. The live object-path shape is
`/Engine/Transient.<EngineClass>:<GameInstance>.<SubsystemClassName>_<index>`
which is non-obvious and varies by engine type (`LyraEditorEngine_0` in
editor PIE, `GameEngine_0` at runtime).

**Proposed RPC** (read-only; extends the existing `system.inspect.*`
namespace rather than spawning a new tree):

| Method | Params | Returns |
|---|---|---|
| `system.inspect.list_subsystems` | `scope: "Engine" \| "Editor" \| "GameInstance" \| "World" \| "LocalPlayer"` (optional; omit for all five) | `{ subsystems: [ { scope, className, objectPath, ownerPath } ] }` |

`scope` echoes which collection the entry came from. `className` is the
leaf C++/BP class name (e.g. `ApiSubsystem`, `AssetEditorSubsystem`).
`objectPath` is `Subsystem->GetPathName()` — directly resolvable by
downstream `property.get` / `object.*` RPCs. `ownerPath` is the
`GetPathName()` of the outer (`UEngine` / `UEditorEngine` /
`UGameInstance` / `UWorld` / `ULocalPlayer`) so the agent can correlate
multi-instance scopes (e.g. one entry per local player).

**World/owner selection rule:** for `World` / `GameInstance` /
`LocalPlayer` scopes, pick `GEditor->PlayWorld` when non-null, otherwise
fall back to `GEditor->GetEditorWorldContext().World()` — same precedent
as sibling ticket `B-inspect-misses-pie-world.md` and
`Handlers/System/SessionsHandler.cpp`'s internal `GetGameInstance()`
helper. `Engine` scope uses `GEngine` directly. `Editor` scope uses
`GEditor`. Outside PIE, the `GameInstance`/`World`/`LocalPlayer` slots
return an empty array (never an exception) when no owner is resolvable.

**Implementation hint** (no compile, no commit):

```cpp
// New handler file under Handlers/System/ (e.g. SubsystemInspectHandler.cpp).
// Each block emits one entry per non-null subsystem with
// { scope, className: S->GetClass()->GetName(),
//   objectPath: S->GetPathName(),
//   ownerPath: Owner->GetPathName() }.

if (Scope.IsEmpty() || Scope == TEXT("Engine"))
{
    if (GEngine)
        for (UEngineSubsystem* S : GEngine->GetEngineSubsystemArray<UEngineSubsystem>())
            Emit(TEXT("Engine"), S, GEngine);
}
if (Scope.IsEmpty() || Scope == TEXT("Editor"))
{
    if (GEditor)
        for (UEditorSubsystem* S : GEditor->GetEditorSubsystemArray<UEditorSubsystem>())
            Emit(TEXT("Editor"), S, GEditor);
}
if (Scope.IsEmpty() || Scope == TEXT("GameInstance"))
{
    if (UGameInstance* GI = ResolvePieFirstGameInstance())
        for (UGameInstanceSubsystem* S : GI->GetSubsystemArrayCopy<UGameInstanceSubsystem>())
            Emit(TEXT("GameInstance"), S, GI);
}
if (Scope.IsEmpty() || Scope == TEXT("World"))
{
    if (UWorld* W = ResolvePieFirstWorld())
        for (UWorldSubsystem* S : W->GetSubsystemArray<UWorldSubsystem>())
            Emit(TEXT("World"), S, W);
}
if (Scope.IsEmpty() || Scope == TEXT("LocalPlayer"))
{
    if (UGameInstance* GI = ResolvePieFirstGameInstance())
        for (ULocalPlayer* LP : GI->GetLocalPlayers())
            for (ULocalPlayerSubsystem* S : LP->GetSubsystemArrayCopy<ULocalPlayerSubsystem>())
                Emit(TEXT("LocalPlayer"), S, LP);
}
```

`UEngine::GetEngineSubsystemArray<T>()`,
`UGameInstance::GetSubsystemArrayCopy<T>()`,
`UWorld::GetSubsystemArray<T>()`, and
`ULocalPlayer::GetSubsystemArrayCopy<T>()` are the canonical accessors;
passing the abstract base type as `T` returns every concrete subclass.

**Acceptance check** (manual, in PIE on `L_Core` with the standard PDS
player flow):
- `system.inspect.list_subsystems { scope: "GameInstance" }` returns one
  entry each for `ApiSubsystem`, `UsersSubsystem`, `TrackCacheSubsystem`
  with `objectPath` ending in
  `B_DroneGameInstance_C_<n>.ApiSubsystem_0` (and the matching shape for
  the other two) and `ownerPath` ending in `B_DroneGameInstance_C_<n>`.
- `property.get` against the returned `ApiSubsystem` `objectPath` reads a
  known `UApiSubsystem` UPROPERTY successfully (sanity-checks the path is
  a real, resolvable UObject ref, not just a label).
- `system.inspect.list_subsystems` with no args returns entries from all
  five scopes; `Engine` and `Editor` scopes are non-empty even outside
  PIE.
- `system.inspect.list_subsystems { scope: "World" }` outside PIE
  resolves to the editor-context world's `UWorldSubsystem` instances
  (e.g. `UnrealEdEngine` editor-side world subsystems) — never an
  unhandled exception.

**Workaround until implemented:** `python.execute` with
`unreal.find_object(gi, 'ApiSubsystem_0')` after manually resolving the
`UGameInstance` via
`unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_game_world().get_game_instance()`.
Functional but requires the agent to already know the leaf instance name
and outer chain, and returns stdout text rather than typed JSON.

## History
- `#1-initial-repro` `OPEN` reporter — No `system.inspect.*` (or any other namespace) handler returns live subsystem instances. Confirmed via `mcp__editor-automation__call path="system.inspect"` — response lists only `find_by_tag` / `list_objects` / `inspect_class` / `find_by_class` / `inspect_object`, all actor-centric over a UWorld. Grep against `Plugins/EditorAutomationRpcGateway/Source/` for `GetSubsystemArray` / `GetEngineSubsystem` / `list_subsystems` / `find_subsystem` hit only `SystemControlHandler.cpp`, `CodeNodeEmitter.cpp`, `BlueprintCreationHandler.cpp`, `EditorAutomationRpcGateway_BlueprintCreationShim.cpp`, `BlueprintComponentHandler.cpp` — all incidental references (BP component subsystem lookups, code-emitter helpers), no RPC handler registration for subsystem enumeration. The only working path today is `python.execute` calling `unreal.find_object(gi, 'ApiSubsystem_0')`, which forces the agent to already know the leaf instance suffix (`_0`/`_1`/…) and the outer's full path — non-obvious shape `/Engine/Transient.<EngineClass>:<GameInstance>.<SubsystemClassName>_<index>` that varies by engine type (`LyraEditorEngine_0` editor-PIE vs `GameEngine_0` runtime). Sister ticket `F-inspect-game-singletons.md` covers GameInstance/GameMode/PC/LP singletons; this ticket covers the orthogonal subsystem dimension (a single GameInstance hosts N GameInstanceSubsystems).
- `#2-add-list-subsystems-handler` `IN-REVIEW` developer — Added `system.inspect.list_subsystems` handler in `Handlers/System/SubsystemInspectHandler.cpp` enumerating Engine/Editor/GameInstance/World/LocalPlayer subsystem arrays via UE's `*ArrayCopy<T>()` accessors; PIE-first world resolution via `McpActorUtils::ResolveQueryWorld`. Regression test `FSubsystemInspectListEditorScopeTest` verifies the Editor scope.
- `#3-verify-pass` `DONE` tester — Verified in live PIE on `L_Core`: `system.inspect.list_subsystems { scope: "GameInstance" }` returns 49 live entries (incl. `ApiSubsystem`, `UsersSubsystem`, `TrackCacheSubsystem`, `DroneSelectionSubsystem`, ...) each with full `objectPath`/`className`/`scope`/`ownerPath`. Each path is directly callable by `property.get` (confirmed against `ApiSubsystem_0.eServerType → "School"`). Acceptance bar met.
