---
id: B-blueprint-mutators-compile-ungated-tick-unsafe
title: "59 registered verbs run FKismetEditorUtilities::CompileBlueprint on the handler's own stack while family K in the tick-unsafe table lists 2 — the shipped editor-kill mechanism rides every structural Blueprint mutation, and 55 of them do not even report the reinstance"
status: IN-REVIEW
severity: Critical
category: bug
tags: [blueprint, safepoint, tick-gate, reinstance, editor-crash, dispatch, shared-editor, multi-agent, coverage-gap, scs, networking, game-framework]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Family K gates the two verbs the kills came through, not the mechanism

`E-compile-reinstances-live-instances-no-guard` fixed `blueprint.compile` and
`blueprint.set_default` and named the rest as a known gap ("~40 other call sites reach
`FKismetEditorUtilities::CompileBlueprint` ... They now REPORT via the shared path but are
not safe-point gated and do not refuse"). Counted against source, the gap is **59 registered
verbs**, and the half of that sentence about reporting is wrong: **55 of the 59 report
nothing at all.**

## The mechanism is not verb-specific

`Dispatch/SafePoint.cpp:479-521` (family K) and
`Handlers/Blueprint/BlueprintReinstancingGuard.h:1-67` establish the chain once:
`FKismetEditorUtilities::CompileBlueprint` flushes the compilation manager's reinstancing
queue (`BlueprintCompilationManager.cpp:392` -> `FlushReinstancingQueueImpl` at `:2086`) ->
`ReplaceInstancesOfClass_Inner` -> `World->EditorDestroyActor(OldActor, bShouldModifyLevel=true)`
(`KismetReinstanceUtilities.cpp:3011`). `FTickTaskLevel` cooked one
`TGraphTask<FTickFunctionTask>` per enabled tick function at StartFrame holding a RAW
`FTickFunction*` (`TickTaskManager.cpp:284`), so the actor is destructed under a queued task
that still calls the PURE_VIRTUAL `FTickFunction::ExecuteTick` (`EngineBaseTypes.h:524`).

Nothing in that chain reads the method name. Every call below is
`CompileBlueprint(BP)` with default options on the handler's own stack, and the transport
marshals every request with `AsyncTask(ENamedThreads::GameThread, ...)`, so
`PinWrightSafePoint::IsSafeNow()` reads false for all of them — the exact condition
family K exists for. The shipped kill was `blueprint.compile` on `/Game/FPS/AI/BP_EnemyCharacter`
with an instance placed in another stream's open map, no PIE. `blueprint.scs.add_component`
or `networking.set_property_replicated` on that same Blueprint would have killed the editor
identically.

## The ungated verbs

**`blueprint.*` — 27.** `Handlers/Blueprint/`:
`add_variable` (`BlueprintPropertyHandler.cpp:181`), `remove_variable` (`:323`),
`rename_variable` (`:390`), `set_variable_metadata` (`:676`), `set_variable_settings` (`:921`),
`add_event` (`BlueprintEventHandler.cpp:374`), `remove_event` (`:699`),
`add_function` (`BlueprintFunctionHandler.cpp:328`, `:435`), `add_macro` (`:712`),
`set_function_settings` (`:973`), `remove_function` (`:1094`, `:1132`),
`add_dispatcher` (`BlueprintDispatcherHandler.cpp:138`),
`add_interface` / `remove_interface` (`BlueprintInterfaceHandler.cpp:198`, one shared body),
`reparent` (`BlueprintReparentHandler.cpp:77`),
`delete_unused_variables` (`BlueprintVariableCleanupHandler.cpp:121`),
`graph.delete_orphaned_nodes` (`BlueprintGraphOrphanHandler.cpp:171` in `DeleteOrphans`, called
at `:317` and `:368`), `modify_scs` (`BlueprintComponentHandler.cpp:522`, `:710`),
`compile_bpir` (`BpirCompilerHandler.cpp:242`, `:255`, `:299`),
`insert_bpir_at_node` (`:469`, `:512`), `insert_bpir_before_node` (`:647`, `:690`),
`scs.duplicate_component` (`SCSComponentDuplicateHandler.cpp:336`), and the five SCS verbs that
share `FSCSHandlers::FinalizeBlueprintSCSChange` (`PinWright_SCSHandlers.cpp:55`):
`scs.add_component` (called `:876`), `scs.remove_component` (`:984`),
`scs.reparent_component` (`:1186`), `scs.set_transform` (`:1298`), `scs.set_property` (`:1586`).

**`networking.*` — 8.** `NetworkingHandler.cpp`: `set_property_replicated` (`:292`),
`set_replication_condition` (`:341`), `create_rpc_function` (`:550`),
`configure_rpc_validation` (`:612`), `set_rpc_reliability` (`:664`),
`set_autonomous_proxy` (`:740`), `set_replicated_using` (`:956`),
`add_network_prediction_data` (`:1103`).

**`game_framework.*` — 7.** `GameFrameworkHandler.cpp`: `configure_game_rules` (`:345`),
`configure_round_system` (`:422`), `configure_team_system` (`:496`),
`configure_scoring_system` (`:566`), `configure_spawn_system` (`:633`),
`set_respawn_rules` (`:762`), `configure_spectating` (`:817`).

**`interaction.*` — 6**, all through `CompileAndGetDefaultObject` (`InteractionHandler.cpp:98`):
`configure_interaction_trace` (`:353`), `configure_interaction_widget` (`:418`),
`add_interaction_events` (`:508`), `configure_door_properties` (`:556`),
`configure_switch_properties` (`:609`), `configure_chest_properties` (`:674`).

**`vehicle.*` — 5.** `ChaosVehicleHandler.cpp`: `create_wheel_asset` (`:287`),
`set_wheel_asset_property` (`:368`), `set_wheel_setup` (`:465`), `remove_wheel_setup` (`:525`),
`set_suspension` (`:618`).

**`ai.*` — 3**, through `AssignObjectDefaultToControllerCDO` (`AIHandler.cpp:143`):
`assign_behavior_tree` (`:341`), `assign_blackboard` (`:392`), `run_behavior_tree` (`:2587`).

**`gas.*` — 2.** `GASHandler.cpp`: `add_attribute` (`:913`),
`create_execution_calculation` (`:2651`). **`widget.bind_event` — 1**
(`WidgetEventBindingHandler.cpp:76`).

## The reporting claim in the parent ticket does not hold

`reinstanced` is emitted only by `BlueprintHandlerUtils::AddCompileDiagnosticsToJson`
(`BlueprintHandlerUtils.cpp:507-541`), which has exactly **five** non-test callers:
`BlueprintCompileHandler.cpp:56` (`blueprint.compile`, already gated),
`BlueprintDispatcherHandler.cpp:153`, `BlueprintFunctionHandler.cpp:339`,
`BlueprintInterfaceHandler.cpp:222`. So **4 of the 59 ungated verbs report; 55 do not.**

Two distinct reasons, both worth fixing separately:

- ~45 of the call sites never touch `CompileBlueprintWithDiagnostics`
  (`BlueprintHandlerUtils.cpp:410`) at all — they call `FKismetEditorUtilities::CompileBlueprint`
  directly, so `FBlueprintCompileDiagnostics::Reinstanced` is never populated for them.
- The three BPIR verbs DO call `CompileBlueprintWithDiagnostics`
  (`BpirCompilerHandler.cpp:255`, `:299`, `:512`, `:690`) but hand-roll their JSON from the
  struct's `Errors`/`Warnings`/`bCompiled`/`Status` fields (`:299-313`, `:511-526`) and never call
  `AddCompileDiagnosticsToJson`, so the survey they paid for is computed and dropped.

`blueprint.compile_bpir` carries a third wrinkle: it pre-compiles Widget Blueprints directly at
`BpirCompilerHandler.cpp:242` **before** the survey at `:255`/`:299` is taken, so even wired up
its report would describe the world after its own first reinstance.

## Severity

Rubric impact class is `Critical` on its own terms — editor crash, twice shipped, in a
shared editor where one death costs every attached stream its unsaved work
(`E-compile-reinstances-live-instances-no-guard` history `#2`). Reach then argues up, not
down: the two verbs family K gates are not the ones agents call most.
`blueprint.scs.add_component`, `blueprint.add_variable`, `blueprint.compile_bpir` and
`blueprint.modify_scs` are the everyday Blueprint authoring surface, and
`blueprint.set_default`'s own doc already tells callers to reach for
`blueprint.add_variable`/`blueprint.compile` first. Reach cannot bump past `Critical`, so
`Critical` it is. Board precedent for the same shape at the same rating:
`B-niagara-compile-while-live-component-vectorvm-assert` (Critical — compile with a live
instance kills the editor). `B-compile-material-not-tick-gated` is the precedent for the
ticket shape itself: one verb reaching a documented hazard family and missing from the table.

Amplifier worth stating: `game_framework.configure_*` compiles once **per variable** through
`SetVariableDefaultValue` -> `SetBPVarDefaultValueGF` (`GameFrameworkHandler.cpp:184`, `:49`).
`configure_round_system` writes six defaults (`:385-415`) plus its own compile at `:422`, so a
single request flushes the reinstancing queue seven times.

## Not affected, stated so nobody re-files them

- `Compiler/BpirCompiler.cpp:3243`, `:3328` pass `EBlueprintCompileOptions::RegenerateSkeletonOnly`
  — skeleton regeneration, not the full compile that flushes the reinstancing queue.
- `GameFrameworkHandler.cpp:119` (`CreateGameFrameworkBlueprint`) is unreachable: its only caller
  is `GF_CREATE_CLASS_HANDLER`, defined at `:261` and never instantiated anywhere in the tree.
- `blueprint.add_construction_script` (`BlueprintCompileHandler.cpp:63`) and
  `blueprint.scs.set_spline_points` (`SCSHandler.cpp:276`) mutate without compiling.
- The table route is legal for all 59: `FRpcDispatcher::DispatchMethod` is never called with any
  of these names (the only `blueprint.*` cross-dispatch in the tree is `blueprint.create`,
  `BlueprintInfoHandler.cpp:330`), and no handler body among them opens its own
  `AsyncTask(ENamedThreads::GameThread, ...)`. This is the same check family K, I and J each make
  before choosing a table entry over an in-handler gate.

**Workaround:** before any structural Blueprint mutation, make sure no loaded world holds an
instance of the class — close the map that has one, or run the edit against a Blueprint nothing
is placed from. `blueprint.compile` with `allowReinstancing` omitted is currently the only verb
that will tell you: it refuses with `LIVE_INSTANCES_WOULD_BE_REINSTANCED` naming the count and
the owning worlds, so calling it first is a usable pre-flight for the other 59.

**Fix:** the parent's two halves, applied to the rest.

- POSITION: declare all 59 tick-unsafe. Adding 59 hand-written entries to family K is the shape
  that produced this ticket; prefer landing `E-tick-unsafe-declared-at-registration` first and
  annotating each handler at its `REGISTER_RPC_HANDLER`, which also gets the source-scan ratchet
  that keeps the 60th verb from shipping ungated. If that ticket is not taken, the table entries
  are still correct and still the right stopgap — do not hand-write per-handler
  `IsSafeNow()`/`DeferToSafePoint` splits, per the rule at `Dispatch/SafePoint.h:341-362`.
- CONSENT, and it is cheaper than it looks: route the ~45 direct
  `FKismetEditorUtilities::CompileBlueprint` calls through `CompileBlueprintWithDiagnostics`
  (already the shared path, already surveys before compiling) and emit through
  `AddCompileDiagnosticsToJson`, which turns reporting on for all of them with no per-site survey
  code. Wire the three BPIR verbs into `AddCompileDiagnosticsToJson` instead of their hand-rolled
  field writes, and move `compile_bpir`'s Widget pre-compile at `:242` behind the same survey.
  Then adopt `RefuseIfLiveInstancesWouldBeReinstanced` + `allowReinstancing` on the structural
  mutators — at minimum the `blueprint.scs.*` family, `blueprint.modify_scs`,
  `blueprint.reparent`, `blueprint.compile_bpir` and the BPIR inserts, which are the ones that
  rebuild placed actors people are looking at.

## Fix

The finding was TRUE. All 59 registered verbs resolved from the full-compile call sites were
absent from the tick-unsafe table even though the same reinstancing queue flush already justified
the two existing Blueprint entries. The position fix adds the complete verified verb set to
`GTickUnsafeMethodNames`; registration-time declaration was deliberately not introduced because
it would require a broader registry migration while the existing table provides complete gating
with no cross-dispatch exceptions for this family.

Every non-test handler full compile now routes through
`BlueprintHandlerUtils::CompileBlueprintWithDiagnostics`; the only remaining direct production
calls are that chokepoint and the two explicitly allow-listed `RegenerateSkeletonOnly` BPIR
compiler calls. The three BPIR verbs now preserve the pre-Widget-compile survey and emit it, and
the structural minimum named by the ticket — all six compiling `blueprint.scs.*` verbs,
`blueprint.modify_scs`, `blueprint.reparent`, `blueprint.compile_bpir`, and both BPIR inserts —
advertises `allowReinstancing`, refuses before mutation without consent, and reports the same
pre-compile survey on success. All 59 routes now emit the shared compile result whenever they
actually compile: `compiled`, `status`, their established error/warning array names, and
`reinstanced` when the pre-compile survey found live instances. Conditional or idempotent paths
that do not compile keep their existing response shape without synthetic compile fields.

Files changed:
- `Source/PinWright/Private/Dispatch/SafePoint.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintReinstancingGuard.h`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintReinstancingGuard.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BpirCompilerHandler.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/SCSHandler.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/SCSComponentDuplicateHandler.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintComponentHandler.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintReparentHandler.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintVariableCleanupHandler.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphOrphanHandler.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintPropertyHandler.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintEventHandler.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintFunctionHandler.cpp`
- `Source/PinWright/Private/Handlers/AI/AIHandler.cpp`
- `Source/PinWright/Private/Handlers/Interaction/InteractionHandler.cpp`
- `Source/PinWright/Private/Handlers/Networking/NetworkingHandler.cpp`
- `Source/PinWright/Private/Handlers/Physics/ChaosVehicleHandler.cpp`
- `Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`
- `Source/PinWright/Private/Handlers/Systems/GameFrameworkHandler.cpp`
- `Source/PinWright/Private/Handlers/UI/WidgetEventBindingHandler.cpp`
- `Source/PinWright/Private/PinWright_SCSHandlers.h`
- `Source/PinWright/Private/PinWright_SCSHandlers.cpp`
- `Source/PinWright/Private/Tests/Infra/TestHandlerTickSafetyRatchet.cpp`
- `Source/PinWright/Private/Tests/Bpir/TestBpirCompilePreexistingErrorsRepair.cpp`
- `docs/wiki-src/blueprint.md`
- `docs/wiki-src/blueprint.scs.md`
- `docs/wiki-src/vehicle.md`

Test id: `PinWright.infra.tick_safety.HandlerHazardsStayGated`. The shared ratchet verifies all
59 method names remain tick-unsafe, holds direct compile calls to the diagnostics chokepoint and
the two skeleton-only exceptions by explicit file/count allow-list, verifies each exception's
compile option, requires the structural consent sites, and resolves every one of the 59 registered
handler bodies to either a direct diagnostics emitter or one of the two explicitly checked shared
reporting seams. Per the worker brief, no editor, build, MCP call, or automation run was performed.
`E-tick-unsafe-declared-at-registration` remains OPEN and unchanged. Consent and response-shape
migration remain intentionally separate: consent is limited to the ticket's eleven required
structural mutators, while reporting now covers all 59 compile routes. Registration-time safety
metadata remains the broader follow-up; the complete table is the accepted stopgap here.

A verifier follow-up closed two response holes in that migration. All three BPIR
`BLUEPRINT_COMPILE_FAILED` paths now retain the already-populated compile/reinstance payload, and
`vehicle.set_suspension` aggregates every distinct wheel Blueprint compile into both the existing
top-level diagnostics and a per-asset `compileResults` array; any failed compile now makes the RPC
fail with `COMPILE_FAILED` instead of returning success from whichever result happened
to be retained.

## History
- `#1-counted-59-ungated-verbs` `OPEN` reporter — Filed from the known-gap note in `E-compile-reinstances-live-instances-no-guard`'s Fix section and `Dispatch/SafePoint.cpp:512-519`. Verified by source reading, no editor run. Counted every non-test `FKismetEditorUtilities::CompileBlueprint(` call site in `Plugins/PinWright/Source` and resolved each to its registered verb, following four shared helpers to their callers (`FSCSHandlers::FinalizeBlueprintSCSChange`, `InteractionHandler::CompileAndGetDefaultObject`, `AIHandler::AssignObjectDefaultToControllerCDO`, `GameFrameworkHandler::SetBPVarDefaultValueGF`): 59 registered verbs across 8 namespaces, all full compiles, none safe-point gated, none refusing. Corrected the parent's reporting claim — `AddCompileDiagnosticsToJson` has five non-test callers, so only `blueprint.add_function`, `blueprint.add_dispatcher`, `blueprint.add_interface` and `blueprint.remove_interface` emit `reinstanced`; the other 55 report nothing, and the three BPIR verbs compute the survey and discard it. Confirmed no cross-dispatch caller for any of the 59 and excluded four false positives (two `RegenerateSkeletonOnly` compiles, one dead call site behind an uninstantiated macro, two non-compiling verbs). Checked the family for duplicates — `B-compile-material-not-tick-gated` (different verb family), `B-niagara-compile-while-live-component-vectorvm-assert` (Niagara, DONE), `B-bpir-macro-recompile-orphans-caller-instances` (macro identity, DONE), `B-safepoint-tick-gate-inert-on-simpletickobjects-path` (the gate's own predicate, DONE), `B-compile-bpir-edit-lost-during-pie` (save, not reinstancing) — none covers this; filed NEW.
- `#2-gated-all-blueprint-compile-mutators` `IN-REVIEW` developer — Added all 59 verified verbs to the dispatcher tick-unsafe family, routed every handler full compile through `CompileBlueprintWithDiagnostics`, adopted the existing live-instance guard on the ticket's eleven structural mutators, and added the shared `PinWright.infra.tick_safety.HandlerHazardsStayGated` source ratchet. Kept registration-time declarations and the other 48 verbs' consent/response migrations outside this fix. Static source and diff checks only; no Unreal process, build, MCP call, or automation run.
- `#3-completed-compile-response-migration` `IN-REVIEW` developer — Completed the ticket's reporting half for all 59 routes: every actual full compile now emits `AddCompileDiagnosticsToJson` directly or through an explicitly checked shared reporter, while conditional no-compile paths remain unchanged. Extended the ratchet to bind every registered route to that reporting contract and kept `allowReinstancing` refusal at the requested eleven structural mutators. Static source and diff checks only; no Unreal process, build, MCP call, or automation run.
- `#4-preserved-all-failure-diagnostics` `IN-REVIEW` developer — Retained the populated diagnostics payload on all three BPIR Blueprint-compile failures and changed `vehicle.set_suspension` to return per-asset plus aggregate diagnostics and fail when any touched wheel Blueprint fails compilation. Extended the existing BPIR behavior coverage and shared structural ratchet. Static checks only; no Unreal process, build, MCP call, or automation run.
