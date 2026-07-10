---
id: B-open-asset-world-map-load-crash
title: "editor.open_asset on a World hard-crashes the editor — routes maps through OpenEditorForAsset → Map_Load, tripping !LevelList.Contains(TickTaskLevel) during outgoing-world teardown"
status: IN-REVIEW
severity: Critical
category: bug
tags: [editor, open_asset, world, map-load, editor-crash, cold-load, ticktaskmanager, open-asset-world-crash]
encounters: 1
lastSeen: 2026-07-10T21:38:22.4092205+03:00
claimedBy: fuzz2
claimedAt: 2026-07-10T21:56:53.0570649+03:00
---

# `editor.open_asset` on a World hard-crashes the editor (TickTaskManager assertion during Map_Load teardown)

## Symptom

Opening a World/level asset through `editor.open_asset` fatally crashes the
editor. A cold-booted editor (PinWright subsystem up, empty-args MCP index
returned fine) was pointed at the saved map `/Game/Maps/ExampleProjectWelcome`
via `editor.open_asset` and died with an engine assertion:

```
Assertion failed: !LevelList.Contains(TickTaskLevel)
[File:D:\build\++UE5\Sync\Engine\Source\Runtime\Engine\Private\TickTaskManager.cpp] [Line: 1987]
=== Critical error: ===

FDebug::CheckVerifyFailedImpl2
FTickTaskManager::FreeTickTaskLevel          (TickTaskManager.cpp:1987)
ULevel::~ULevel                              (Level.cpp:466)
FObjectPurge::DestroyObjects / IncrementalPurgeGarbage
CollectGarbage
UEditorEngine::Cleanse
UEditorEngine::EditorDestroyWorld
UEditorEngine::Map_Load                      (EditorServer.cpp:2464)
UAssetDefinition_World::OpenAssets
UAssetEditorSubsystem::OpenEditorForAsset
UnrealEditor-PinWright.dll!AutoHandler_318_  (EditorCommandHandler.cpp:408)
```

The `UnrealEditor.exe` process then exited (post-open check: NO_PROCESS) and the
MCP transport dropped mid-call: "Editor not reachable at
http://127.0.0.1:24966/mcp ... ([WinError 10054] remote host forcibly closed the
connection)". Asset class is `/Script/Engine.World`.

## Root cause

`editor.open_asset` (`Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp:373-420`)
routes **every** asset type — including Worlds — through the generic
`OpenEditorForAsset` path with no special-casing for `UWorld`:

```cpp
  UObject *Asset = UEditorAssetLibrary::LoadAsset(AssetPath);   // line 402
  if (!Asset) {
    Ctx.SendError(TEXT("LOAD_FAILED"), TEXT("Failed to load asset"));
    return true;
  }

  const bool bOpened = AssetEditorSS->OpenEditorForAsset(Asset); // line 408 — crash site
```

For a World, `UAssetEditorSubsystem::OpenEditorForAsset` dispatches to
`UAssetDefinition_World::OpenAssets`, which performs a full
`UEditorEngine::Map_Load` (the same thing double-clicking a `.umap` does). That
`Map_Load` first tears down and GCs the currently-loaded world
(`EditorDestroyWorld` → `Cleanse` → `CollectGarbage` → `~ULevel` →
`FreeTickTaskLevel`), and the outgoing world's level is still registered in the
tick-task manager's `LevelList`, tripping the `!LevelList.Contains(TickTaskLevel)`
assertion.

This host makes the crash acute: `/Game/Maps/ExampleProjectWelcome` is the
editor **startup map** (`Config/DefaultEngine.ini`:
`EditorStartupMap=/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome`), so a
cold-booted editor already has that exact world live. `editor.open_asset` on it
forces `Map_Load` to reopen the already-active map, tearing down the live world
and hitting the assertion during that GC.

## What should happen

`editor.open_asset` should not be able to hard-crash the editor. Two directions
for the fixer to weigh (the guilty line, EditorCommandHandler.cpp:408, is common
to both):

1. **open_asset should special-case Worlds.** A `UWorld` target should route
   through the map-load-with-proper-teardown path (the same resolution
   `editor.open_level` uses — its dedicated sibling verb for maps) or be rejected
   with a clean error steering the caller to `editor.open_level`, rather than
   handed to the generic `OpenEditorForAsset` that Map_Loads over the live world.
   Re-opening the *already-active* map in particular should be a no-op, not a
   destroy-and-reload.
2. **Save-time integrity angle.** This surfaced through the workflow's
   cold-restart CorruptionCheck on a map the attempt had just re-lit and saved
   (`level.save` → `saved:true`). The save path has no World-level integrity gate
   analogous to the Blueprint one, so if the lighting mutations left the world's
   level tick-registration inconsistent, nothing at save time would catch it. The
   fixer should determine whether the tick-task inconsistency is inherent to
   `Map_Load`-over-the-active-map (hypothesis 1, expected to reproduce on a
   pristine map too) or specific to the saved world state (hypothesis 2).

The crash callstack points primarily at hypothesis 1: the assertion fires in the
teardown of the **outgoing** world during `Map_Load`, before the saved map's data
is deserialized into the editor world — so the trigger is the open/Map_Load path,
not (yet) the saved map's on-disk contents.

## Repro (confirmed by a real cold editor restart)

1. Attempt re-lit and saved `/Game/Maps/ExampleProjectWelcome`
   (`lighting.spawn_light`, `lighting.ensure_single_sky_light`,
   `lighting.configure_shadows`, `lighting.setup_global_illumination`,
   `lighting.setup_volumetric_fog`, `lighting.set_ambient_occlusion`,
   `lighting.set_exposure`, then `level.save` → job `completed`, `saved:true`).
2. Editor quit + relaunched headless WITHOUT baseline restore. Cold boot was
   clean: PinWrightSubsystem initialized, empty-args MCP index returned.
3. `editor.open_asset {assetPath:"/Game/Maps/ExampleProjectWelcome"}` →
   editor crashed with the assertion above; process exited; MCP transport
   dropped ([WinError 10054]).

(The restart is itself the replay-confirmation; not re-replayed through the MCP
to avoid re-crashing the editor.)

severity rationale: impact=corruption/editor-crash × reach=every-session (opening
a level asset is a routine action; here it is also the startup map) -> Critical

## History
- `#2-fix-route-world-to-open-level` `IN-REVIEW` developer — GO (hypothesis 1; severity Critical unchanged, full scope, no splits). Decision: `editor.open_asset` must special-case `UWorld` — detect the World target and cross-dispatch to the dedicated `editor.open_level` verb (which delegates to `level.load`, no-oping via `DoesRequestedLevelMatchCurrentWorld` when the requested map is already active) instead of handing it to the generic `OpenEditorForAsset -> Map_Load` that destroys-and-reloads the live world. This fixes only the crashing self-reload case and preserves the working different-map load through the vetted level.load path; hypothesis 2 (World save-time integrity gate) deliberately NOT pursued (ruled out as the trigger — the assertion fires in outgoing-world teardown, not from saved bytes). Implementation + compile/test verification to follow.
- `#1-initial-repro` `OPEN` reporter — Cold-restart CorruptionCheck crashed the freshly cold-booted editor when `editor.open_asset` opened the saved startup World `/Game/Maps/ExampleProjectWelcome`. Fatal assertion `!LevelList.Contains(TickTaskLevel)` (TickTaskManager.cpp:1987) fires in `FTickTaskManager::FreeTickTaskLevel` during `~ULevel` GC inside `UEditorEngine::Map_Load` teardown, dispatched from `editor.open_asset` (EditorCommandHandler.cpp:408, `AutoHandler_318_`). Confirmed via source read: `editor.open_asset` routes ALL asset types through `OpenEditorForAsset`, which for a World does a full `Map_Load` that tears down the live world; the target here is also the `EditorStartupMap`, so the cold editor already had that world loaded and open_asset forced a destroy-and-reload. Leading hypothesis is that `editor.open_asset` must special-case Worlds (route to the map-load path like `editor.open_level`, no-op on the active map, or reject) rather than Map_Load over the live world; secondary hypothesis is a missing World save-time integrity gate. Distinct from `B-open-level-engine-mount-mangled` (that is `editor.open_level` path-string mangling, no crash) and from the Widget-Blueprint cold-load corruption in `B-bp-saved-state-corruption-mcp-edits` (different asset type + crash signature).
