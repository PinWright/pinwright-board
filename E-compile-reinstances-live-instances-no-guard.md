---
id: E-compile-reinstances-live-instances-no-guard
title: "blueprint.set_default / blueprint.compile reinstance live PIE instances with no warning"
status: OPEN
severity: Medium
category: ergonomic
tags: [blueprint, set-default, compile, reinstance, pie, live-instances, safety]
encounters: 2
lastSeen: 2026-09-03T04:49:49Z
---

# `blueprint.set_default` / `blueprint.compile` reinstance live PIE instances with no warning

`blueprint.set_default` finalizes by calling `FKismetEditorUtilities::CompileBlueprint`
(`Source/PinWright/Private/Handlers/Blueprint/BlueprintPropertyHandler.cpp:485`, added by
sibling `B-blueprint-set-default-not-persisted` so CDO writes persist), and `blueprint.compile`
calls it directly. A full compile flushes the reinstancing queue
(`FBlueprintCompilationManagerImpl::FlushReinstancingQueueImpl` → `ReplaceInstancesOfClass_Inner`
→ `UWorld::EditorDestroyActor`), which **tears down every live instance of the class — including
actors in a running PIE session** — then re-creates them. Neither handler detects that PIE is
active / live instances exist, and neither warns that the call will destroy and re-spawn them.
A blind agent calling "set a default value" has no signal it just triggered a heavyweight
reinstancing pass over live PIE state.

Reinstancing-on-compile is normal, designed UE behavior and is harmless when actor teardown is
well-behaved; a human compiling a BP during PIE from the editor UI gets the identical reinstance.
It only faults when a game's teardown path is fragile. Session evidence: `set_default` of a
material default on `/App/HELIOS/Drones/Atlas/B_PioneerSumo` during a live PIE with drone
instances → compile → reinstance → `EditorDestroyActor` → (game) `ADrone::RefreshControllerChanged`
→ `UFPVCameraComponent::IsLocallyControlled` null-deref → editor ACCESS_VIOLATION.
**Scope: the null-deref was the crash's proximate cause and is fixed game-side — this ticket is
only the MCP-side gap: no warning/opt-out for reinstance-on-compile during live PIE.** Board
precedent for surfacing dangerous editor state: `B-editor-save-all-pie-diagnostic` (surface
`pieActive`) and the material open-editor guard family (`B-material-graph-edit-clobbered-by-open-editor`).

**Workaround:** Set CDO defaults / compile before starting PIE, or stop PIE first
(`ui.stop_play`); don't run `set_default` / `compile` on a class that has live PIE instances.
**Fix:** Detect a running PIE / live instances of the target class (`GEditor->PlayWorld` +
instance scan) and either return a `pieActive` / `reinstancedLiveInstances` warning field on the
success response, or gate the compile behind an explicit opt-in arg — default warn-not-block,
since reinstancing is legitimate hot-reload; a hard block would break valid compile-during-PIE.

## History
- `#1-initial-repro` `OPEN` reporter — Verified in source: `blueprint.set_default`'s finalize block calls `FKismetEditorUtilities::CompileBlueprint(Blueprint)` at `BlueprintPropertyHandler.cpp:485` with no PIE / live-instance guard or warning; `blueprint.compile` does the same. Session repro: `blueprint.set_default` (material default) on `/App/HELIOS/Drones/Atlas/B_PioneerSumo` in an editor running PIE with live drone instances triggered the compile → reinstancing queue flush → `ReplaceInstancesOfClass_Inner` → `UWorld::EditorDestroyActor` destroyed the live drones mid-call → game-code null-deref (`ADrone::RefreshControllerChanged` → `UFPVCameraComponent::IsLocallyControlled`) → editor ACCESS_VIOLATION. The game null-deref is fixed separately; filed here for the un-warned reinstance-on-compile side effect during live PIE. Checked the family — `B-bp-saved-state-corruption-mcp-edits`, `B-material-graph-edit-clobbered-by-open-editor`, `B-material-graph-mutators-bypass-editor-open-guard`, `B-bpir-macro-recompile-orphans-caller-instances`, `B-compile-save-after-compile-timeout`, and sibling `B-blueprint-set-default-not-persisted` — none cover compile-during-live-PIE reinstancing of live actors; filed NEW.

- `#2-editor-killed-with-no-pie-and-across-stream-boundaries` `OPEN` reporter - **Second encounter kills the editor, and it breaks two of this ticket's framing assumptions. Recommending severity Medium -> High and category ergonomic -> bug.** 2026-09-03 04:49:34 UTC, UE 5.8, shared editor on port 27145 with seven agent streams working.

**(a) No PIE was running.** The ticket is scoped to "live PIE instances". PIE had shut down almost three minutes earlier (`LogPlayLevel: Shutting down PIE online subsystems` at 04:46:46). The crash happened in the ordinary editor world tick. A live instance in the plain editor world is sufficient; the PIE qualifier is not part of the precondition.

**(b) The fault is a dangling tick function inside the engine, not a game-side null-deref.** This ticket's first encounter recorded a game-side `IsLocallyControlled` null-deref and scoped itself to "the MCP-side gap: no warning". Here the crash is `LowLevelFatalError [EngineBaseTypes.h:524] Pure virtual not implemented ()` reached through `TGraphTask<FTickFunctionTask>::ExecuteTask` -> `FTickTaskSequencer::ReleaseTickGroup` -> `FTickTaskManager::RunTickGroup` -> `UWorld::Tick` (`LevelTick.cpp:1750`) -> `UEditorEngine::Tick`. The reinstanced actor's `FTickFunction` was still registered and queued when the old object was trashed, so the tick task dispatched into a vtable that no longer exists. No game code is on the stack. That is an engine-level teardown ordering fault the reinstancing pass exposes, and it means "harmless when actor teardown is well-behaved" does not hold — nothing in this project's `BP_EnemyCharacter` teardown is involved.

Sequence, straight from `Saved/Logs/EAContentExamples58.log`:

```
04:46:46:057  LogPlayLevel: Shutting down PIE online subsystems      <- PIE ends
04:47:41:912  Cmd: MAP LOAD FILE=".../Content/FPS/Test/T_Weapons.umap"
04:49:34:008  LogBlueprint: Compiling Blueprint '/Game/FPS/AI/BP_EnemyCharacter'
04:49:34:234  LogUObjectHash: Compacting FUObjectHashTables data took 1.48ms
04:49:34:237  LowLevelFatalError: Pure virtual not implemented ()
```

**(c) The multi-agent shape is the part that generalises, and it is new.** The compile was issued by the AI stream against its own Blueprint. The live instance was `WPN_EnemyRifleTest`, a placed `BP_EnemyCharacter` in `T_Weapons` — the WEAPONS stream's level, open at the time because WEAPONS had just loaded it. So **one stream compiled a Blueprint and killed the editor through an instance sitting in a different stream's level.** Neither party could have seen it coming: the compiling agent had no reason to know another team's map holds an instance of its class, and the map's owner had no reason to know someone was about to compile. This is not a discipline problem that a working agreement can fix — a `does any loaded world contain an instance of this class` check belongs in the verb.

Cost: one editor death taking every stream's in-flight work, the fourth of this session and the second distinct root cause (the other three were `render.capture_open_level`'s hit-proxy `ColorRT` assert, `B-capture-open-level-hitproxy-colorrt-assert-kills-editor`).

**Fix, sharpened by (b):** a warning is not sufficient. Before flushing the reinstancing queue, the handler should enumerate live instances of the class across all loaded worlds and either unregister their tick functions first or refuse with the count and the owning world named. Reporting `instancesReinstanced` and the worlds they were in would also let a caller tell a no-op compile from one that just rebuilt another team's level actors.
