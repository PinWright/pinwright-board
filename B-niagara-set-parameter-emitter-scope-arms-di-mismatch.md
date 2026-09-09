---
id: B-niagara-set-parameter-emitter-scope-arms-di-mismatch
title: "niagara.set_parameter on an emitter-scoped rapid-iteration store flips a system from dataInterfaceCheck 'consistent' to 'mismatched', one emitter per write, arming the VectorVM assert — the response reports the state it just created but never says the write caused it"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, set-parameter, rapid-iteration, data-interface, vectorvm, dataInterfaceCheck, latent-corruption, editor-kill, spawnRapidIteration, updateRapidIteration, vfx]
encounters: 1
lastSeen: 2026-09-07T07:45:00Z
related: [B-niagara-di-count-mismatch-vectorvm-assert-kills-editor, B-module-input-override-inert-runtime-uses-rapid-iteration-default, B-niagara-force-compile-resets-rapid-iteration-values]
---

# A parameter write turns a green system red, one emitter at a time

`niagara.set_parameter` with `scope: "spawnRapidIteration"` or `"updateRapidIteration"` (the two
emitter-scoped stores) causes the target **emitter's** particle scripts to resolve their data
interfaces. The compiled bytecode of a system that has not compiled in this session carries **zero**
`DataInterfaceInfo`, so the fresh resolve produces 1-3 entries against 0 compiled and the system
flips to `dataInterfaceCheck: "mismatched"` — the state
`B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` records as an editor-killing `appError`
on the next tick.

The write reports `success: true` and echoes `dataInterfaceCheck: "mismatched"`, but nothing
distinguishes "this system was already broken" from "this call just broke it". A caller who did not
validate immediately before the write cannot tell.

`scope: "systemUpdateRapidIteration"` (and the other system-wide scopes) does **not** do this.

## Measured, live editor port 27145, UE 5.8, EAContentExamples58, 2026-09-07

`/Game/FPS/VFX/NS_Impact_Concrete`, five emitters, freshly loaded, nothing else touching it.

```
niagara.validate {assetPath, level:"strict"}
# -> dataInterfaceCheck: "consistent", valid: true, errors: []

niagara.set_parameter {scope:"spawnRapidIteration", emitter:"Flash",
    name:"Constants.Flash.InitializeParticle.Color", type:"Color", value:{r:7,g:5,b:3,a:1}}
# -> success, dataInterfaceCheck: "mismatched"
#    mismatchedScripts: [Flash.SpawnScript 0/1, Flash.UpdateScript 0/1]

niagara.set_parameter {scope:"systemUpdateRapidIteration",
    name:"Constants.Chunks.SpawnBurst_Instantaneous.Spawn Count", type:"int32", value:9}
# -> success, mismatchedScripts UNCHANGED — Chunks is NOT added

niagara.set_parameter {scope:"spawnRapidIteration", emitter:"Light", ...}
# -> mismatchedScripts: [Flash x2, Light x2]
niagara.set_parameter {scope:"spawnRapidIteration", emitter:"Sparks", ...}
# -> mismatchedScripts: [Flash x2, Light x2, Sparks x2]
niagara.set_parameter {scope:"spawnRapidIteration", emitter:"Dust", ...}
# -> mismatchedScripts: [Flash x2, Light x2, Sparks x2, Dust.SpawnScript 0/2, Dust.UpdateScript 0/2]

niagara.validate {assetPath, level:"strict"}
# -> dataInterfaceCheck: "mismatched", valid: false,
#    errors: [NIAGARA_DATA_INTERFACE_MISMATCH x8]
```

The list grows **only** for emitters that received an emitter-scoped write, in the order they were
written. A control system nobody touched (`/Game/FPS/VFX/NS_Impact_Wood`) stayed `consistent`
through all of it, and flipped the same way as soon as its own first emitter-scoped write landed.

Reproduced on all ten systems that took emitter-scoped writes this session:
`NS_Impact_Concrete`, `_Metal`, `_Wood`, `_Dirt`, `_Glass`, `_Water`, `NS_Muzzle_AR`,
`NS_Muzzle_Pistol`, `NS_ShellEject`, `NS_Smoke_Grenade`. Ten for ten.

## Why the "consistent" it starts from is not a real pass

The pre-write `consistent` is `0 compiled == 0 resolved`: nothing had resolved yet. So the verdict
was vacuously true and the write is what forced the comparison to become meaningful. That does not
make the resulting state harmless — the resolved set is now populated and the bytecode's is not,
which is exactly the crash precondition — but it does mean the underlying stale-bytecode condition
predates the write. The defect this ticket reports is that a **mutating verb silently moves a system
into the flagged state and reports the flag without attributing it**.

## Recovery that works, and does not cost the values

`niagara.compile {force: false, wait: true}` clears it. On these systems `force: false` **does** issue
a compile (the emitter scripts are at `compileStatus: null`, so the engine does not consider them
current) and completes in 40-650 ms, after which `dataInterfaceCheck` is `consistent` again and
`valid: true`. `niagara.remove_orphan_data_interfaces` would also close the count gap but by
**deleting** the resolved curve entries and pruning `CachedDefaultDataInterfaces`, which throws away
the `Scale Alpha` / `FloatFromCurve` plumbing the graph does want — not the right repair here.

Ordering that survives everything (see `B-niagara-force-compile-resets-rapid-iteration-values`):

1. all emitter-scoped `set_parameter` writes,
2. **one** `niagara.compile {force:false, wait:true}`,
3. all system-scoped `set_parameter` writes (the compile resets those, not the emitter ones),
4. `asset.save`, never compiling again.

## Impact

High. In a shared editor, an agent doing an ordinary parameter write leaves behind a system that
asserts inside the VectorVM on a background worker the next time anything ticks it — killing the
editor for every other agent, minutes later, with nothing in the crash naming the write. The
`success: true` and the absence of any warning in the response are what make it expensive.

## Fix directions

- Have `set_parameter` re-resolve **and** reconcile, or refuse to leave the system mismatched:
  either request the compile itself, or return a typed warning naming the scripts it just moved.
- Distinguish the two verdicts the field currently conflates: `consistent (0 == 0, nothing resolved)`
  is not the same answer as `consistent (n == n, compared)`. `unverified` already exists for the
  first; `validate` and every verb echoing `dataInterfaceCheck` should use it.
- Report the **delta**: if the write changed the verdict, say so in the response.

## History

- `#1-filed` `OPEN` reporter — `niagara.set_parameter` on `spawnRapidIteration` / `updateRapidIteration` flips a Niagara system from `dataInterfaceCheck: "consistent"` to `"mismatched"`, adding exactly the written emitter's Spawn/Update scripts to `mismatchedScripts` (0 compiled vs 1-3 resolved) while a `systemUpdateRapidIteration` write on the same system in the same batch adds nothing. Measured on `/Game/FPS/VFX/NS_Impact_Concrete` with `niagara.validate {level:"strict"}` before (`consistent`, `valid:true`, no errors) and after (`mismatched`, `valid:false`, 8 x `NIAGARA_DATA_INTERFACE_MISMATCH`), with an untouched control system staying green until its own first emitter-scoped write; reproduced 10/10 across the `/Game/FPS/VFX/` package. That is the precondition `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` records as a VectorVM `appError` that kills the shared editor on the next tick, and the response reports the flag without ever saying the call set it. `niagara.compile {force:false, wait:true}` clears it in under a second and preserves the emitter-scoped values, so the working order is emitter-scope writes, one unforced compile, then system-scope writes, then save.

- `#2-delta-reported-verdicts-separated-live-writes-repaired` `IN-REVIEW` developer — Both fix
  directions this ticket names are implemented. The full file list and the engine-semantics argument
  are in `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` `#11`; this change is the same
  one, read from this ticket's side.
  **The mechanism, from engine source rather than inferred.** An emitter-scoped `set_parameter`
  reaches `NiagaraEdit::NotifyNiagaraObjectChanged`
  (`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditTypes.cpp`), which calls
  `FVersionedNiagaraEmitterData::InvalidateCompileResults()` whenever the target resolved an emitter
  — and that resets each of that emitter's scripts' `FNiagaraVMExecutableData`
  (`NiagaraScript.cpp:3895`). The compiled count drops to 0 while the system's serialized resolved
  set keeps its 1-3 entries. `systemUpdateRapidIteration` leaves `Target.EmitterData` null, which is
  exactly why this ticket's control write adds nothing. So the priming is not the engine resolving
  data interfaces against stale bytecode; it is the compiled side being emptied under an unchanged
  resolved side — which is also why a value-identical no-op write re-arms it.
  **Attribution.** Every mutating verb now measures the verdict before the mutation
  (`NiagaraEdit::RecordDataInterfaceVerdictBefore`, called from `ModifyResolvedTarget` and
  `NiagaraJsonHelpers::BeginEmitterMutationScope`) and publishes `dataInterfaceDelta {before, after,
  changed, armedByThisWrite, repair, liveInstancesAtRepair}`. `armedByThisWrite` is true only for a
  real pass turning into a mismatch, so an inherited mismatch and an unmeasured before both read
  false rather than claiming authorship the response cannot support.
  **The two verdicts are separated.** `CheckDataInterfaceCounts` returns `Consistent` only when at
  least one compared script still carries compiled results (`FNiagaraVMExecutableData::IsValid()`);
  the 0-against-0 case on invalidated bytecode — the vacuous pass this ticket calls out — now reads
  `unverified`. A script that genuinely owns no data interfaces still reads `consistent`, so the
  change does not flag DI-free systems as unrunnable.
  **The write no longer leaves a live system armed.** When the delta says this write armed it and
  something holds a live instance, `FinalizeNiagaraEdit` quiesces and compiles before returning
  instead of only refusing the save. Narrow on purpose: with nothing live, the batch order this
  ticket documents (emitter-scope writes, one unforced compile, system-scope writes, save) is
  unchanged and is not charged a compile per edit.
  Tests: `PinWright.niagara.data_interface_delta.MutationReportsBeforeAndAfter` drives an
  emitter-scoped `niagara.set_parameter` on a fixture system and asserts the response carries the
  before/after verdict, that `after` agrees with `dataInterfaceCheck`, and that the system is not
  left mismatched; `PinWright.niagara.data_interface_delta.NoOpWriteIsReportedAndNotLeftArmed` is the
  value-identical repeat from `#10` of the sibling ticket
  (`Source/PinWright/Private/Tests/Niagara/TestNiagaraDataInterfaceDelta.cpp`). Counterfactual:
  remove `NiagaraEdit::AddDataInterfaceDelta` from `MakeMutationResult` and `dataInterfaceDelta` is
  absent, so the first assertion of both tests fails and the response is back to echoing a flag it
  cannot attribute.
