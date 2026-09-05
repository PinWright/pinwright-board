---
id: E-compile-reinstances-live-instances-no-guard
title: "blueprint.set_default / blueprint.compile reinstance live PIE instances with no warning"
status: IN-REVIEW
severity: High
category: bug
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

## Fix

Severity Medium -> High and category ergonomic -> bug, per encounter #2's own recommendation:
the second encounter is an editor kill with no PIE and no game code on the stack.

**Ticket verified TRUE against source.** `blueprint.compile` reached
`CompileBlueprintWithDiagnostics` -> `FKismetEditorUtilities::CompileBlueprint`
(`BlueprintCompileHandler.cpp:40`, `BlueprintHandlerUtils.cpp:430`) and `blueprint.set_default`
called it directly (`BlueprintPropertyHandler.cpp`, the finalize block), with no live-instance
check, no report, and no safe-point gating on either verb.

**Root cause, and encounter #2's suggested fix does not work.** Two independent causes:

1. POSITION. The transport marshals every request with `AsyncTask(ENamedThreads::GameThread, ...)`,
   so a handler runs from a named-thread pump inside the engine frame (`PinWrightSafePoint::IsSafeNow`
   reads false there). `FTickTaskLevel` cooks one `TGraphTask<FTickFunctionTask>` per enabled tick
   function at StartFrame and the task holds a RAW `FTickFunction*` (`TickTaskManager.cpp:284`) that
   `DoTask` dereferences into the PURE_VIRTUAL `FTickFunction::ExecuteTick` (`EngineBaseTypes.h:524`).
   Reinstancing trashes the actor mid-frame and the cooked task runs against a destructed object.
   **Unregistering the tick functions first (the ticket's suggestion) does NOT fix this:**
   `FTickTaskLevel::RemoveTickFunction` (`TickTaskManager.cpp:1807`) only edits the manager's lists;
   the already-cooked graph task keeps the same raw pointer. **`EBlueprintCompileOptions::SkipReinstancing`
   is also ruled out** - the engine asserts `ensure(!bSkipReinstancing); // This is an internal option,
   should not go through CompileSynchronouslyImpl` (`BlueprintCompilationManager.cpp:365`).
2. CONSENT. Reinstancing rebuilds placed actors in *any* loaded world and dirties their level
   (`EditorDestroyActor(OldActor, bShouldModifyLevel=true)`, `KismetReinstanceUtilities.cpp:3011`),
   including another agent's open map. Nothing reported it.

**Design.** Both halves, deliberately separate - the safe-point gate cannot consent on the caller's
behalf and the precondition cannot move a stack:

- POSITION: `blueprint.compile` and `blueprint.set_default` added to the existing tick-unsafe method
  table (`Dispatch/SafePoint.cpp`, new family K with the engine chain). Reuses the plugin's own
  gate rather than inventing one; the table route is legal here because neither verb is reached
  through `FRpcDispatcher::DispatchMethod`. Cost: one 0.1s subsystem tick, response unchanged.
- CONSENT: new `BlueprintReinstancingGuard` - survey live instances (derived classes included; CDOs,
  archetypes, EditorPreview and Inactive worlds excluded), grouped by owning world package name.
  `CompileBlueprintWithDiagnostics` surveys *before* the compile and stores it on
  `FBlueprintCompileDiagnostics`, so **every** verb on the shared diagnostics path reports
  `reinstanced {count, actorCount, pieActive, worlds[]}` with zero call-site churn. Emitted only when
  non-empty, so untouched compiles keep their response shape. The refusal is the opt-in gate:
  `LIVE_INSTANCES_WOULD_BE_REINSTANCED` naming the count and each owning world, overridable with
  `allowReinstancing: true`, applied to the two verbs the shipped kills came through.

**Known gap, stated rather than hidden:** ~40 other call sites reach
`FKismetEditorUtilities::CompileBlueprint` (rest of Handlers/Blueprint plus Networking, AI, Physics,
Interaction, SCS). They now REPORT via the shared path but are not safe-point gated and do not refuse.
The right fix is declaring tick-unsafety at `REGISTER_RPC_HANDLER` instead of in a hand-maintained
table; that is a separate change. Noted in the family-K comment.

### Files changed

- NEW `Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/BlueprintReinstancingGuard.h`
- NEW `Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/BlueprintReinstancingGuard.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Dispatch/SafePoint.cpp` (family K, 2 table entries)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.h` (survey on
  `FBlueprintCompileDiagnostics`)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp` (survey
  before compile; emit in `AddCompileDiagnosticsToJson`)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/BlueprintCompileHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/BlueprintPropertyHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Handlers/ErrorCodes.h`
  (`ERR_LIVE_INSTANCES_WOULD_BE_REINSTANCED`)
- NEW `Plugins/PinWright/Source/PinWright/Private/Tests/Blueprint/TestBlueprintReinstancingGuard.cpp`
- `Plugins/PinWright/docs/wiki-src/blueprint.md`

Not compiled and not run - a separate compile pass follows.

### Reviewer verification

1. Build; then run `PinWright.blueprint.reinstancing_guard.*` + `PinWright.blueprint.compile.*`
   (4 new tests: survey counts/empties per world; empty survey adds no field while a populated one
   emits `reinstanced` and names both worlds; dispatcher-level refuse-then-opt-in on a real spawned
   instance; both verbs declare `allowReinstancing` AND appear in `GetTickUnsafeMethods()`).
   Also run `PinWright.core.error_codes.*` and `PinWright.core.safe_point.*` (registry + table
   contracts) and `PinWright.blueprint.set_default.PersistsThroughCompile` (transient BP, no
   instances - must still pass unrefused).
2. Live editor, cross-stream repro: open a map holding a placed actor of some BP, then
   `call("blueprint.compile", {path: "<that BP>"})`. Expect `LIVE_INSTANCES_WOULD_BE_REINSTANCED`
   naming that map. Re-issue with `allowReinstancing: true`: expect success carrying
   `reinstanced.worlds[0].world == <that map package>`, the actor rebuilt, and **no editor death** -
   the pre-fix crash was in the tick immediately after the compile.
3. Confirm a compile of a BP with no live instances is unchanged: no refusal, no `reinstanced` field.
- `#3-verified-in-fps-build` `IN-REVIEW` AI-stream - Direct route retried after the 2026-09-05 plugin pull; **the guard exists, but I could not fire it**. `blueprint.compile_bpir` now documents and accepts `allowReinstancing` (`boolean`, default `false`): "Proceed even though loaded worlds hold live instances of this class. A compile flushes the Blueprint reinstancing queue, which DESTROYS and re-creates every live instance and dirties the level that owns it ... Defaults to false: the call is refused with `LIVE_INSTANCES_WOULD_BE_REINSTANCED` naming the instance count and each owning world." That is this ticket's ask, and it closes the mechanism that cost this checkout an editor: a `blueprint.compile` on `/Game/FPS/AI/BP_EnemyCharacter` reinstanced an instance another stream had placed in an open map, and the editor died at 04:49:34Z with `LowLevelFatalError: Pure virtual not implemented` in a tick-function task inside `UWorld::Tick` (no PinWright frames - the dangling tick function, not the RPC). **Not verified live**: every compile this session ran against an open world holding **zero** instances of the class (`ExampleProjectWelcome`, `T_VFX`, `FPS_Compound`, each checked with `get_all_actors_of_class` immediately before the call), so the refusal path never triggered and I have no first-hand `LIVE_INSTANCES_WOULD_BE_REINSTANCED` payload to paste. Someone should exercise it with a map open that actually holds an instance. One thing worth stating in the wiki: `compile_bpir` compiles implicitly, so the guard has to cover every BPIR write and not just explicit `blueprint.compile` calls - that implicitness is what made the original crash easy to walk into. Left IN-REVIEW for the tester.
