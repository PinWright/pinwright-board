---
id: F-niagara-remove-emitter
title: "Cannot remove an emitter handle from a Niagara system"
status: DONE
severity: Critical
category: feature
tags: [niagara, system, emitter, lifecycle, authoring]
---

# Cannot remove an emitter handle from a Niagara system

`niagara.add_emitter` attaches an existing `UNiagaraEmitter` to a
`UNiagaraSystem` as a new handle. There is **no symmetrical
`niagara.remove_emitter`** to detach a handle. Confirmed by
`call("niagara.remove_emitter")` (returns NOT FOUND) and by grep over the
plugin source — no handler registers a remove-emitter method.

Adjacent ops exist (`add_emitter`, `add_renderer` / `remove_renderer`,
`add_event_handler` / `remove_event_handler`, `add_simulation_stage` /
`remove_simulation_stage`); the missing remove is an asymmetry, not a
design intent.

## Why it matters

System composition is the most common Niagara editing task: drop in an
emitter, see how it looks, swap it for a different one. Without a remove
RPC, the agent cannot:

1. Undo a wrong `add_emitter` — the only path is to leave the bad handle
   in the system and call `set_stack_enabled` on every module to silence
   it. That still leaves the renderer attached and the handle visible in
   the asset.
2. Reduce an existing system — every "this system has too many emitters,
   trim it" workflow is blocked.
3. Re-author from a copied template — duplicate, then prune unused
   emitters is a normal authoring pattern.

`niagara.set_property` with `target.kind: emitterHandle` can edit
properties on the handle (e.g. `bIsEnabled`) but cannot remove the array
element; the generic property reflection path does not handle
`TArray<FNiagaraEmitterHandle>` element removal.

## Workaround

None inside MCP today. Editor UI: select handle in the System Overview
and delete. Or `python.execute` calling
`UNiagaraSystem::RemoveEmitterHandlesById` — bypasses the typed-RPC
surface and the read-first workflow.

## Proposal

Add `niagara.remove_emitter(systemPath, emitter, compile?, save?)` where
`emitter` accepts either the handle name (string) or the handle id
(Guid string), matching the resolver pattern already used by
`niagara.remove_event_handler` (id-or-index) and `niagara.set_property`
(target.kind=emitterHandle).

Implementation surface: `UNiagaraSystem::RemoveEmitterHandlesById`
(NIAGARA_API), then `KillSystemInstances` on the system view-model cache
so any open editor reflects the change. Reuse the session-scoped
`FNiagaraSystemViewModel` cache established by
`F-niagara-event-handler-simstage-authoring`.

```
niagara.remove_emitter(
    systemPath: string,
    emitter: string,            // handle name OR handle id (Guid)
    compile?: boolean,
    save?: boolean
) -> { removed: true, emitter: string, handleId: string, remainingEmitters: number }
```

Edge cases to handle:

- Handle name collision across emitters: prefer Guid; if name is passed
  and multiple match, error `AMBIGUOUS_EMITTER_HANDLE`.
- Last emitter in system: succeeds and leaves system with empty
  `EmitterHandles`. `niagara.create_system` already returns this state.
- Open editor reflection: kill instances so the system overview rebuilds.

## Cross-ref

- `F-niagara-event-handler-simstage-authoring` — same architectural
  pattern (id-or-index resolver, system view-model cache).
- `niagara.add_emitter` — the asymmetric counterpart.

## History
- `#1-no-remove-emitter` `OPEN` reporter — Confirmed via `call("niagara.remove_emitter")` and grep over plugin source that no remove-emitter RPC exists. Asymmetric to existing `add_emitter` and to the add/remove pairs for renderers, event handlers, simulation stages, parameters, data interfaces, and modules. Blocks any system-composition workflow that needs to detach or replace an emitter handle. Proposes `niagara.remove_emitter(systemPath, emitter, compile?, save?)` accepting handle name or Guid, mirroring the existing event-handler / simulation-stage remove pattern. Implementation via `UNiagaraSystem::RemoveEmitterHandlesById` plus the session view-model cache.
- `#2-reviewed-and-confirmed` `OPEN` reporter — Re-verified by grep over `Source/`: `niagara.add_emitter` registers at `NiagaraHandler.cpp:37`, no handler registers `niagara.remove_emitter`. Sibling `remove_renderer` / `remove_event_handler` / `remove_simulation_stage` all present, confirming the asymmetry. `UNiagaraSystem::RemoveEmitterHandlesById` is `NIAGARA_API` in UE 5.6 (`NiagaraSystem.h:341`) so the proposed implementation surface is reachable. Workaround paths confirmed unavailable: `asset.delete` operates on standalone assets (an emitter handle lives inside `UNiagaraSystem::EmitterHandles`, it is not a standalone asset), and `property.set` / `niagara.set_property` are reflection-based and do not support `TArray<FNiagaraEmitterHandle>` element removal. Severity Critical upheld — system composition is core Niagara authoring and there is no in-MCP recovery from a wrong `add_emitter`. API shape `(systemPath, emitter, compile?, save?)` matches the established `remove_event_handler` / `remove_simulation_stage` resolver pattern. No duplicate board entry. Ticket accepted as-is.
- `#3-implemented-remove-emitter` `IN-REVIEW` developer — Added `niagara.remove_emitter` to `NiagaraHandler.cpp` after `add_emitter`; resolver tries Guid then name with ambiguity check; lightweight idiom mirroring add_emitter (no SVM acquisition). Tests in `TestNiagaraRemoveEmitter.cpp` exercise the engine API and dispatcher registration.
- `#4-crash-during-create-system` `OPEN` tester — Crashed: live verification called `niagara.create_system` for `/Game/App/McpVerify/NS_McpVerifyTemp_FNiagaraRemoveEmitter_20260515`; editor exited with `EXCEPTION_ACCESS_VIOLATION` after `UNiagaraSystem::AddEmitterHandle()` hit a missing `GraphSource` on `DefaultEmitter` in `NiagaraHandler.cpp:313`, so `niagara.remove_emitter` could not be exercised.
- `#5-create-system-factory-init` `IN-REVIEW` developer — Kept niagara.remove_emitter and repaired the returned verifier blocker by switching niagara.create_system / niagara.create_emitter to Niagara editor factory initialization, removing the unsafe default-emitter AddEmitterHandle path, and strengthening the create_system regression test so live verification can create an empty system, add a valid emitter, then remove it.
- `#6-verify-fix` `DONE` tester — Verified: live RPC sequence `niagara.create_system` (NS_McpVerifyTemp_FNiagaraRemoveEmitter) → `niagara.create_emitter` (NE_McpVerifyTemp_FNiagaraRemoveEmitter) → `niagara.add_emitter` (handle "TestHandle", emitterId BC31DE5242E2C34E7254FEA97B06723B, emitterCount=1) → `niagara.remove_emitter` (emitter="TestHandle") returned `{removed:true, handleId:BC31DE5242E2C34E7254FEA97B06723B, remainingEmitters:0}` matching the proposed response shape exactly. No editor crash on create_system this time, confirming the factory-init fix. Schema check via `niagara.remove_emitter?` shows the documented params (systemPath, emitter, compile?, save?). Temp assets cleaned via `asset.delete`.
