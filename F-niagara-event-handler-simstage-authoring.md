---
id: F-niagara-event-handler-simstage-authoring
title: "Niagara event handlers, simulation stages, and data interface add/remove are read-only"
status: DONE
severity: Medium
category: feature
tags: [niagara, authoring, event-handler, simulation-stage, data-interface]
---

# Niagara advanced features: inspectable but not authorable

Three Niagara features are fully captured in the IR (read path) but
have **no MCP edit RPC** for add / remove / reorder / mutate
operations:

1. **Event handlers** (`FNiagaraEventScriptProperties`)
   - Inspectable via `niagara_emitters.json` +
     `niagara_model.json`'s `advancedFeatures.eventHandlers`.
   - No `niagara.add_event_handler`, `niagara.remove_event_handler`,
     or `niagara.set_event_handler_property` RPCs.
   - Per-property edits via `niagara.set_property` with
     `target.kind=emitterData` only work if the field is at the
     emitter property root, not inside the event handler array.

2. **Simulation stages** (`UNiagaraSimulationStageBase`)
   - Inspectable via `niagara_emitters.json::simulationStages` +
     `niagara_model.json`'s `advancedFeatures.simulation_stage`.
   - No `niagara.add_simulation_stage` /
     `niagara.remove_simulation_stage` /
     `niagara.set_stage_property` RPCs.
   - The generic `set_property` cannot manipulate TArray elements,
     so even reading stage properties via the generic path doesn't
     allow editing.

3. **Data interfaces** add / remove
   - Per-DI properties are editable via
     `niagara.set_property { target.kind: dataInterface, target.index: N }`.
   - No way to **add** a new DI to an emitter (e.g. attach a
     skeletal mesh DI for surface sampling) or **remove** one.
   - Blocks any emitter-from-scratch authoring workflow.

## Why these matter

Event handlers and simulation stages are the two primary
"advanced" emitter features:
- Event handlers wire emitter-to-emitter signaling
  (collision events, death events, generic gameplay events).
- Simulation stages enable GPU compute iterations and
  data-interface-driven simulation (e.g. flocking on a render
  target).

Data interfaces are how Niagara talks to the rest of the engine
(skeletal meshes, splines, render targets, audio, gameplay
attributes). Authoring an effect from scratch typically requires
adding at least one DI.

Without these RPCs, MCP can describe these features faithfully but
cannot create or modify them.

## Fix scope

Six new RPCs plus two new `target.kind` resolvers on the existing
reflection-based `niagara.set_property` (per-element property edits
keep using `set_property`; only add / remove of the containing
element gets dedicated RPCs).

```
niagara.add_event_handler(assetPath, emitter, source?, executionMode?, sourceEventName?, spawnNumber?, maxEventsPerFrame?, bRandomSpawnNumber?, compile?, save?)
  -> { eventHandlerId, eventHandlerIndex }
niagara.remove_event_handler(assetPath, emitter, eventHandlerId | eventHandlerIndex, compile?, save?)

niagara.add_simulation_stage(assetPath, emitter, stageClass?, atIndex?, compile?, save?)
  -> { stageId, stageIndex }
niagara.remove_simulation_stage(assetPath, emitter, stageId | stageIndex, compile?, save?)

niagara.add_data_interface(assetPath, scope, parameterName, dataInterfaceClass, emitter?, compile?, save?)
niagara.remove_data_interface(assetPath, scope, parameterName, emitter?, compile?, save?)

# extend existing niagara.set_property:
target.kind: "eventHandler"     with target.index OR target.entryId (UsageId Guid)
target.kind: "simulationStage"  with target.index OR target.entryId (UsageId Guid)
```

Per-DI property edits keep using the existing
`target.kind: dataInterface` path — no change.

Implementation goes through `UNiagaraEmitter::AddEventHandler` /
`RemoveEventHandlerByUsageId` / `AddSimulationStage` /
`RemoveSimulationStage` (all `NIAGARA_API`),
`FNiagaraEmitterViewModel::AddEventHandler(props, /*bResetGraphForOutput=*/true)`
(NIAGARAEDITOR_API) for the script-source wiring, and a session-scoped
`FNiagaraSystemViewModel` cache that reuses any open editor SVM. The
engine-private `FNiagaraStackGraphUtilities::ResetGraphForOutput` is
vendored locally for the simulation-stage path because it lacks
`NIAGARAEDITOR_API` export.

## History
- `#1-initial-spec` `OPEN` reporter — Niagara coverage audit confirmed event handlers, simulation stages, and data interface add/remove are fully inspectable but not authorable. Per-property `set_property` works for sub-fields but cannot add/remove the containing element. Required for emitter-from-scratch authoring and any GPU-compute or event-driven effect modification.
- `#2-reshape-and-implement` `IN-REVIEW` developer — Reshaped from 9 RPCs to 6 RPCs + 2 `target.kind` resolvers (`eventHandler`/`simulationStage`) on `niagara.set_property`. Implemented six `niagara.{add,remove}_{event_handler,simulation_stage,data_interface}` in `NiagaraAdvancedEditHandler.cpp` plus session-scoped `NiagaraSystemViewModelCache` and vendored `NiagaraGraphResetUtils::ResetGraphForOutput` (engine helper not exported). Five regression tests cover round-trip add/remove for all three families plus the two new resolver kinds.
- `#3-verify-advanced-authoring` `DONE` tester — Verified: duplicated `/Game/Effects/Particles/Environmental/NS_CharacterDash` to `/Game/App/UI/Test/NS_McpVerifyTemp_F_niagara_event_handler_simstage_authoring`, inspected emitter `DashPoints`, then ran `niagara.add_event_handler` (`eventHandlerIndex:0`, id `EA0CDFA248B0CBEAA9A94C92EA4328CB`), `niagara.set_property` with `target.kind:eventHandler` on `SpawnNumber`, `niagara.add_simulation_stage` (`stageIndex:0`, id `FA05ABE9457C1AA6462C33BDCADAC4A2`, class `/Script/Niagara.NiagaraSimulationStageGeneric`), `niagara.set_property` with `target.kind:simulationStage` on `bEnabled`, `niagara.add_data_interface` for user parameter `McpVerifyDI` using `/Script/Niagara.NiagaraDataInterfaceCurve`, and the matching `remove_event_handler`, `remove_simulation_stage`, and `remove_data_interface`; all mutation responses returned `success:true`, compile/save disabled, and cleanup left the temp asset with `existsAfter:false`.
