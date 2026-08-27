---
id: B-niagara-compile-wait-does-not-wait
title: "niagara.compile returns compiled:true in ~10ms without waiting; wait:true is ignored and status says 'requested'"
status: OPEN
severity: High
category: bug
tags: [niagara, compile, async, silent-noop, race, corrupts-saved-asset, data-interface-mismatch, editor-crash]
encounters: 2
lastSeen: 2026-08-27T20:35:00+05:00
---

# `niagara.compile` acknowledges a request and calls it a result

`niagara.compile` declares a `wait` parameter. With `wait: true` it still returns immediately, and
its own response contradicts itself: `compiled: true` next to `status: "requested"`.

```
niagara.compile {assetPath: "/Game/Atlantis/VFX/NS_Plankton_Drift", force: true, wait: true}
-> {"compiled": true, "durationMs": 8.130103349685669, "status": "requested"}
```

Observed `durationMs` across this session: 4.67, 8.13, 10.92, 17.18, 25.54, 39.74, 42.46 ms. A real
Niagara system compile does not complete in tens of milliseconds, and the engine log proves it did
not: the matching completions land seconds to a minute later.

## Evidence — `Saved/Logs/EAContentExamples58.log`

```
[2026.08.27-13.29.46:714][898] LogNiagara: Compiling System NiagaraSystem
    /Game/Atlantis/VFX/NS_Plankton_Drift.NS_Plankton_Drift took  0.089178 sec (time since issued).
[2026.08.27-13.41.31:097][322] ... took  0.886659 sec (time since issued).
[2026.08.27-13.45.08:596][141] ... took  0.080630 sec (time since issued).
[2026.08.27-13.47.17:506][334] ... took 50.368397 sec (time since issued).
```

The `50.368397 sec` line is the one that settles it. The RPC that issued that compile returned
`compiled: true` in under 40 ms; the compile itself finished **50 seconds later**. Nothing in the
response distinguishes the two.

Second half of the same defect: a `niagara.compile` **without** `force` on a system the editor
considers up to date emits no `Compiling System` line at all — no compile ran — and still returns
`compiled: true`. Between 13:32 and 13:41 this session, five such calls produced zero compile lines.

## Impact

High, and it is a race rather than a hard failure, so it reproduces intermittently. The documented
authoring loop is edit -> compile -> verify, and every verification path that follows a compile —
`niagara.spawn_actor`, `effect.activate_niagara`, `effect.advance_simulation`,
`render.capture_open_level` — can execute against the *previous* compiled scripts while the caller
believes the compile finished. The symptom is a capture that shows the pre-edit behaviour, which
reads as "my edit did not apply" and sends the author hunting the wrong bug. It cost this run several
capture cycles on `/Game/Atlantis/VFX/NS_Plankton_Drift` before the log timestamps explained it.

It also undermines `niagara.validate`: validate run right after compile can report `NCS_UpToDate`
from the prior compile.

## Repro

```
niagara.set_module_input {...}                     // any change that requires a recompile
niagara.compile {assetPath: <system>, force: true, wait: true}
// -> compiled:true, status:"requested", durationMs ~10-40
```
Then `grep -n "Compiling System .*<SystemName>" Saved/Logs/EAContentExamples58.log` and compare the
completion timestamp against the RPC's own return time.

## Workaround

Treat `compiled: true` as "queued". Either poll the log for the `Compiling System ... took` line, or
insert unrelated round trips before spawning/capturing. Passing `force: true` at least guarantees a
compile is *issued*; without it the call can be a complete no-op that still reports success.

## Suggested fix

- Make `wait: true` actually block on completion (the editor already publishes it — `UNiagaraSystem`
  exposes `HasOutstandingCompilationRequests()` / `WaitForCompilationComplete()`, and
  `niagara.validate` already reports `hasOutstandingCompilationRequests` and `hasActiveCompilations`,
  so the state is reachable from the same handler).
- Until then, stop reporting `compiled: true` for a request that was only queued. Report
  `compiled: false, status: "requested"` and let `status` be the single truth, or rename the field.
  `compiled: true` beside `status: "requested"` is the contradiction callers act on.
- Report `issued: false` when no compile was actually needed/started, instead of `compiled: true`.

---

# This is not only a reporting defect: `compile:true, save:true` on one call SAVES an invalidated compile, and the asset then asserts in the VectorVM (encounter 2, 2026-08-27)

Because the compile is asynchronous and the same handler call performs the save immediately
afterwards, the save runs while the compile is still in flight. `UNiagaraScript`'s presave then
finds the script's **compiled** data-interface list empty while the emitter **resolves** two, logs

```
LogNiagara: Warning: Data interface count mismatch during script presave.
            Invaliding compile results (see full log for details).
            Script: /Game/Atlantis/VFX/NS_FishSchool.NS_FishSchool:Fountain.UpdateScript
LogNiagara: Compiled DataInterfaceInfos:
LogNiagara: Resolved DataInterfaceInfos:
LogNiagara:   Name:Emitter.VectorField32,       Type: NiagaraDataInterfaceVectorField, ...
LogNiagara:   Name:Emitter.Scale Alpha.FloatCurve, Type: NiagaraDataInterfaceCurve,   ...
```

and writes the invalidated result to the `.uasset`. **0 compiled vs 2 resolved.** The asset on disk
is now in a state where the next thing that re-ticks it executes the VM against a mismatched
data-interface table and asserts on a worker thread, killing the editor for everyone
(`B-niagara-di-count-mismatch-vectorvm-assert-kills-editor`, and the same assert reported from the
authoring side as `B-niagara-compile-while-live-component-vectorvm-assert`).

Nothing in the `set_module_input` / `set_property` / `add_module` response says any of this. Each
returns `compiled: true, saved: true`.

## Measured, both directions, same asset, same session

`/Game/Atlantis/VFX/NS_FishSchool`, built by rewriting a duplicate of
`/Niagara/DefaultAssets/DefaultSystem`:

| what was done | `Data interface count mismatch` in the log? |
|---|---|
| ~25 edits, each `{compile: true, save: true}` | **44 occurrences** across `Fountain.SpawnScript` + `Fountain.UpdateScript` |
| then: `niagara.compile {force:true, wait:true}` as its own call, **then** `asset.save {force:true}` as a separate call | **none** |

Same for the two sibling systems built the same way — `NS_FishSchool_Orange` (4 occurrences before,
0 after) and `NS_FishSchool_Silver` (3 before, 0 after). Log timestamps: last mismatch
`2026.08.27-15.13.56`, clean saves at `15.17.30`, `15.18.49`, `15.19.29`.

## Working recipe until this is fixed

Never pass `compile: true` and `save: true` on the same `niagara.*` edit. Instead:

```
niagara.set_* {..., compile: false, save: true}     // repeat for every edit; save is the crash floor
...
niagara.compile {assetPath, force: true, wait: true} // its own call
asset.save      {assetPath, force: true}             // its own call, after the compile
```

The gap between the two RPCs (one client round trip) is empirically enough for the compile to land
on this project's systems. That is luck, not a guarantee — the underlying `wait` is still not
waiting, which is what this ticket is about.

Cheap disk-level check that the save came out clean, needing no editor:

```
grep -c "Data interface count mismatch" Saved/Logs/EAContentExamples58.log   # before/after the save
```

## Suggested fix, in priority order

1. Make `wait: true` actually block until `UNiagaraSystem::HasOutstandingCompilationRequests()` is
   false (with a timeout and a reported `waited`/`timedOut` field), and make `status` report
   `completed` vs `requested` truthfully.
2. In `FinalizeNiagaraEdit`, when both `bCompile` and `bSave` are requested, wait for the compile
   before saving — or refuse the combination with a typed error naming the ordering, because as it
   stands the pair is a data-corrupting footgun on the most common call shape in the namespace.
3. Independently, presave should refuse to serialise a script whose compiled/resolved DI lists
   disagree rather than writing the invalidated result.


## History
- `#1-initial-repro` `OPEN` reporter — Found while authoring `/Game/Atlantis/VFX/NS_Plankton_Drift`
  and `/Game/Atlantis/VFX/NS_Bubbles_Ambient` on the Atlantis map build. Seven `niagara.compile`
  calls returned `compiled:true` with `durationMs` 4.67-42.46 ms and `status:"requested"`, against
  engine log completions of 0.08-50.37 sec for the same system. Also recorded: non-`force` compiles
  that emitted no `Compiling System` line at all yet still returned `compiled:true`.
- `#2-compile-save-same-call-corrupts-asset` `OPEN` reporter - 2026-08-27, UE 5.8, Atlantis build. Escalation: the async compile is not merely misreported, it corrupts what gets saved. `{compile:true, save:true}` on one `niagara.*` edit saves while the compile is still in flight; presave finds 0 compiled vs 2 resolved `DataInterfaceInfos` (`Emitter.VectorField32` NiagaraDataInterfaceVectorField, `Emitter.Scale Alpha.FloatCurve` NiagaraDataInterfaceCurve), logs `Data interface count mismatch during script presave. Invaliding compile results` and writes the invalidated result to the .uasset - after which anything that re-ticks the system asserts in the VectorVM and kills the editor (`B-niagara-di-count-mismatch-vectorvm-assert-kills-editor`). Measured on `/Game/Atlantis/VFX/NS_FishSchool`: 44 mismatch occurrences while using `{compile:true,save:true}` per edit; **zero** after switching to `niagara.compile {force:true,wait:true}` as its own call followed by `asset.save {force:true}` as a separate call. Reproduced and fixed the same way on `NS_FishSchool_Orange` (4 -> 0) and `NS_FishSchool_Silver` (3 -> 0); last bad save `15.13.56`, clean saves `15.17.30`/`15.18.49`/`15.19.29`. Workaround recipe and a grep-based disk check added above.
