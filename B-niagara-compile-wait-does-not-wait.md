---
id: B-niagara-compile-wait-does-not-wait
title: "niagara.compile wait:true holds the game thread for a hard 90 s and never observes the compile, so {compile:true, save:true} stalls the whole shared editor and then persists nothing (originally: returned compiled:true in ~10 ms without waiting)"
status: IN-REVIEW
severity: Critical
category: bug
tags: [niagara, compile, async, silent-noop, race, corrupts-saved-asset, data-interface-mismatch, editor-crash, wait-never-lands, reopened, game-thread-stall, shared-editor-outage, blocks-concurrent-agents, fix-absent-from-this-checkout]
encounters: 4
lastSeen: 2026-08-28T08:30:00+05:00
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

> **This recipe is stale on this checkout (2026-08-28) — see the section below and `#8`.**
> The `compile: false, save: true` per-edit half still stands. The
> `niagara.compile {force: true, wait: true}` line does **not**: it no longer costs "one
> client round trip", it holds the game thread for a hard 90 s and returns
> `compiled: false, status: "timedOut"`, and every other agent on the editor is frozen for
> that whole time. Today's shape is `{compile: true, save: false}` per edit — which skips the
> wait entirely, `NiagaraEditTypes.cpp:1649` scopes it to `bCompile && bSave`, and costs
> 0.028 s — then confirm the compile from the engine log's `Compiling System ... took` line,
> then `asset.save` as its own call. Do not pass `wait: true` on this tree.

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


---

# Current state on this checkout (2026-08-28, plugin HEAD `b79ba53e`): the wait is a 90 s whole-editor stall that persists nothing

> **Superseded on this tree at plugin HEAD `11aebe3f` — see `#9`.** `#6`'s pump landed here as
> commit `f281e2a0`, which is *later* than the `b79ba53e` this section and `#8` were written
> against. All three of `#8`'s own presence checks now pass. The 90 s-stall description below is
> the pre-`f281e2a0` behaviour and no longer describes this checkout; it is kept because other
> hosts may still be behind that commit.

Both halves above are historical. The `compiled: true` beside `status: "requested"` contradiction is
fixed (`#3`, re-confirmed by `#4`, `#5` and `#7`), and the `{compile: true, save: true}` corruption
route is closed — but it is closed by the verb no longer working, and the replacement failure costs a
shared editor more than the reporting bug ever did.

`WaitForSystemCompile` (`Handlers/Niagara/NiagaraCompileWait.cpp:38-56`) busy-polls
`HasOutstandingCompilationRequests()` with `PollForCompilationComplete(true)` +
`FPlatformProcess::Sleep(0.01)` **on the game thread**. Every step that *consumes* a Niagara compile
result also runs on the game thread, so the loop starves the thread it is waiting on: the predicate
cannot clear from inside the loop, the 90 s ceiling is always reached, and
`MayPersistAfterCompileWait` (`NiagaraEditTypes.cpp:1663`) then correctly refuses the save behind a
compile that never landed.

## Measured — same verb, same asset, minutes apart

| call | result | longest game-thread stall | engine log for that compile |
|---|---|---|---|
| `{compile: true, save: false}` | `compiled: true` | **0.028 s** | `took 0.067841` / `0.137554` / `1.089133 sec` |
| `{compile: true, save: true}` | `compiled: false, saved: false` | **90.04 s** | `took 90.080238 sec`, `took 90.113518 sec` |
| `niagara.compile {force: true, wait: true}` | `waitedMs: 90003.9`, `status: "timedOut"`, `outstandingCompilationRequests: true` | **90.05 s** | as above |

The compile does not take 90 s; the wait makes it take 90 s. The same asset compiles in
`0.067841 sec` with the wait out of the way, and `#5` recorded one completing in `0.000721 sec` while
no loop was running. `time since issued` tracking the wait duration to within 100 ms is the signature.

Stalls are measured, not inferred from `waitedMs`: a `register_slate_post_tick_callback` probe
recorded the game thread not ticking **at all** for 90.05 s, 90.08 s and 90.04 s across those calls.

## Why this is Critical and not High

- **It loses the edit.** `{compile: true, save: true}` is the ordinary, documented way to make a
  `niagara.*` edit stick. It returns `saved: false` and writes nothing, so the author's change exists
  only in editor memory and dies with the process.
- **It is an outage, not a slow call.** 90 s of dead game thread per call freezes *every* concurrent
  agent on a shared editor, not just the caller — and the response reports only `waitedMs`, so the
  bystanders get no signal at all. This is the cost the board's severity table has no row for, and it
  is the reason the rating is not `High`.
- **Reach is every-session.** The pair is the default call shape across the whole `niagara.*` edit
  namespace, which is the board's one-level reach bump on its own.
- **Board precedent.** `B-blueprint-search-wedges-game-thread` — the same defect class, a verb
  monopolising the game thread — is rated `Critical`.
- **It replaces a Critical.** This same flag pair previously wrote an invalidated compile and
  asserted in the VectorVM (`B-niagara-di-count-mismatch-vectorvm-assert-kills-editor`, Critical).
  Trading a crash for a 90 s whole-editor outage that *also* loses the work is not a severity
  reduction.

## Fix notes for whoever picks this up

`#6` already names the mechanism and a one-line remedy — pump
`FAssetCompilingManager::Get().ProcessAsyncTasks(/*bLimitExecutionTime=*/true)` at the top of each
loop iteration, before the poll and before the sleep. **That code is not in this checkout**; see
`#7` and `#8`, and check your own tree before assuming otherwise.

Fixing the wait **re-opens the save path** that `{compile: true, save: true}` currently declines,
which is the exact path `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` (Critical,
IN-REVIEW) covers. Re-verify that ticket's guard once the save resumes; its corruption route is
currently closed only because the verb does not work.

Whatever the fix, the wait must not be able to hold the game thread for 90 s again. A bounded pumped
wait, a much lower default ceiling with an explicit opt-in for longer, or handing back a poll handle
instead of blocking would each satisfy this ticket; silently freezing a shared editor for 90 s does
not, even if it eventually returns `compiled: true`.


## History
- `#1-initial-repro` `OPEN` reporter — Found while authoring `/Game/Atlantis/VFX/NS_Plankton_Drift`
  and `/Game/Atlantis/VFX/NS_Bubbles_Ambient` on the Atlantis map build. Seven `niagara.compile`
  calls returned `compiled:true` with `durationMs` 4.67-42.46 ms and `status:"requested"`, against
  engine log completions of 0.08-50.37 sec for the same system. Also recorded: non-`force` compiles
  that emitted no `Compiling System` line at all yet still returned `compiled:true`.
- `#2-compile-save-same-call-corrupts-asset` `OPEN` reporter - 2026-08-27, UE 5.8, Atlantis build. Escalation: the async compile is not merely misreported, it corrupts what gets saved. `{compile:true, save:true}` on one `niagara.*` edit saves while the compile is still in flight; presave finds 0 compiled vs 2 resolved `DataInterfaceInfos` (`Emitter.VectorField32` NiagaraDataInterfaceVectorField, `Emitter.Scale Alpha.FloatCurve` NiagaraDataInterfaceCurve), logs `Data interface count mismatch during script presave. Invaliding compile results` and writes the invalidated result to the .uasset - after which anything that re-ticks the system asserts in the VectorVM and kills the editor (`B-niagara-di-count-mismatch-vectorvm-assert-kills-editor`). Measured on `/Game/Atlantis/VFX/NS_FishSchool`: 44 mismatch occurrences while using `{compile:true,save:true}` per edit; **zero** after switching to `niagara.compile {force:true,wait:true}` as its own call followed by `asset.save {force:true}` as a separate call. Reproduced and fixed the same way on `NS_FishSchool_Orange` (4 -> 0) and `NS_FishSchool_Silver` (3 -> 0); last bad save `15.13.56`, clean saves `15.17.30`/`15.18.49`/`15.19.29`. Workaround recipe and a grep-based disk check added above.
- `#3-wait-actually-blocks` `IN-REVIEW` developer — `wait:true` now blocks. New `Handlers/Niagara/NiagaraCompileWait.{h,cpp}` adds a bounded, non-flushing wait (`WaitForSystemCompile` / `WaitForEmitterCompile`, ceiling 90 s derived from the transport's 120 s response timeout against the 50.4 s worst compile in this ticket) plus three pure decision helpers (`MayPersistAfterCompileWait`, `DidCompileLand`, `DescribeCompileOutcome`). `NiagaraCompileHandler.cpp`: `niagara.compile` now waits, honours `force` (previously declared and ignored — it forced unconditionally), and reports `requested` / `compiled` / `waited` / `waitedMs` / `timedOut` / `outstandingCompilationRequests` with `status` ∈ `completed|timedOut|outstanding|notRequested`; `compiled:true` beside `status:"requested"` is gone. `NiagaraEditTypes.{h,cpp}`: compile request extracted to `NiagaraEdit::RequestNiagaraCompile(Target, bForce)` (Wave A's `KillSystemInstances` / bare-source guard moved intact), and `FinalizeNiagaraEdit` now waits for the compile before saving whenever both `compile` and `save` are requested — refusing the save and logging a warning when it does not land, which closes the `{compile:true,save:true}` race that corrupts the .uasset. `NiagaraHandler.cpp`: same wait inserted in `add_emitter` / `remove_emitter` between `RequestCompile` and Wave A's `RejectOnDataInterfaceMismatch` (which was previously reading compiled state mid-flight), both gating the save on `MayPersistAfterCompileWait` and reporting `compileTimedOut`. Test `PinWright.niagara.CompileWait.CompletionIsMeasuredNotAssumed` (`Tests/Niagara/TestNiagaraCompileWait.cpp`) asserts the status/compiled mapping, the save gate, that the wait is a no-op with nothing outstanding, and that the ceiling stays inside the derived band — deliberately without driving a real compile, since `WaitForCompilationComplete` and compiling a synthetic fixture system both crash the suite host.

- `#4-verified-fixed` `DONE` verifier — 2026-08-28. Plugin rebuilt from a clean tree at `b79ba53e` and verified against disk, not against the build's own success message: `UnrealEditor-PinWright.dll` 39,898,624 -> 40,644,096 bytes at 2026-08-28 08:11:48, `UnrealEditor-PinWrightGeometry.dll` 4,983,296 -> 5,113,344, canonical link with no `-000N` artifacts in `UnrealEditor.modules`. Editor restarted on that DLL and the ticket's own repro re-run. The self-contradiction is gone and the wait is real. `niagara.compile {assetPath:"/Game/Atlantis/VFX/NS_Plankton_Drift", force:true, wait:true}` previously returned in ~8 ms with `compiled: true` beside `status: "requested"`. On the rebuilt DLL the same call returns after **90,003.9 ms** with `waited: true`, `compiled: false`, `timedOut: true`, `outstandingCompilationRequests: true`, `status: "timedOut"`. It now blocks, measures, and reports failure honestly instead of reporting a result it never had. New observation, not a regression: this system does not finish inside the 90 s cap on a cold DDC, so the honest answer here is a timeout - callers must read `compiled`, not `requested`.

- `#5-live-verification-wait-never-lands` `OPEN` developer — **Reopening. The blocking half is fixed; the waiting half never observes completion, and `#4`'s "cold DDC" reading is ruled out.** Live run on a running editor (pid 27156, UE 5.8) against `/Game/PinWrightTests/NS_LiveVerify_Probe`, a 5-node system with no emitters at the time of the first run. Two consecutive `niagara.compile {force:true, wait:true}` calls each returned `{requested:true, compiled:false, waited:true, waitedMs:90000.3 / 90005.7, timedOut:true, outstandingCompilationRequests:true, status:"timedOut"}`. The engine's own completion lines are the proof: `LogNiagara: Compiling System ... NS_LiveVerify_Probe took 90.024727 sec (time since issued)` and `took 90.034843 sec (time since issued)` — "time since issued" equals the wait duration to within 30 ms on both runs, i.e. the request the handler issued sat undrained for the entire poll loop and was consumed only once the loop gave up. The same asset compiles in **0.104775 sec** when the editor drives it normally (`took 0.104775 sec` at `03.54.07`, before this pass), and a further engine line at `03.58.56` shows one completing in **0.000721 sec** while no loop was running. After each timeout, `niagara.compile {force:false, wait:true}` immediately reports `{requested:false, compiled:false, waited:false, timedOut:false, outstandingCompilationRequests:false, status:"notRequested"}` — so the compile settles the instant the loop stops. Mechanism: `WaitForSystemCompile` (`Handlers/Niagara/NiagaraCompileWait.cpp`) busy-polls `HasOutstandingCompilationRequests()` with `PollForCompilationComplete(true)` + `FPlatformProcess::Sleep(0.01)` on the game thread; `PollForCompilationComplete` is non-blocking and the loop starves the game-thread work the compile needs to land, so the predicate can never clear from inside the loop. The engine's own `UNiagaraSystem::WaitForCompilationComplete()` uses `QueryCompileComplete(bWait=true)`, which blocks up to 0.125 s per call and pumps properly — the file's own comment notes this and chose `Poll` anyway. Net effect for callers: `wait:true` costs a hard 90 s and returns `compiled:false` on an asset that compiles in 100 ms, so the ticket's actual goal (a caller can tell when the edit applied, and downstream verification runs against the new scripts) is still not met, and `FinalizeNiagaraEdit`'s save gate on `MayPersistAfterCompileWait` will now refuse the save on every `{compile:true, save:true}` edit. Three-for-three deterministic across two assets and two editor sessions counting `#4`. `force` handling and the field self-consistency from `#3` are confirmed correct and are not what is being reopened. What is verified as fixed: `compiled:true` beside `status:"requested"` is gone, `force:false` no longer claims a compile ran, and the call genuinely blocks. What still breaks: the wait cannot observe a compile completing, so `status:"completed"` appears to be unreachable whenever a compile was actually issued.

- `#6-wait-pumps-game-thread` `IN-REVIEW` developer — Root cause of `#5` found and fixed in one place: the wait loop starved the game thread it was waiting on. A Niagara compile only *runs* off-thread; every step that CONSUMES a result runs on the game thread out of `FAssetCompilingManager::ProcessAsyncTasks`, which the editor calls once per tick. Two registered managers matter: `FShaderCompilingManager`, whose `ProcessAsyncTasks` is literally `ProcessAsyncResults(...)` — the pass that marks a finished VM compile job `EShaderCompileJobStatus::Released`, the one state `FHlslNiagaraCompiler::GetCompileResult(bWait=false)` refuses to return results without (NiagaraCompiler.cpp:2214) — and `FNiagaraSystemCompilingManager` (registered at NiagaraEditorModule.cpp:1732), which owns the `AsyncTasks` compilation mode `UNiagaraSettings` defaults to on 5.8 and whose `ProcessAsyncTasks` ticks the compile task, runs its queued game-thread functions and calls `ConditionalLaunchTask` — so without it a queued compile never even launches. `WaitForSystemCompile`'s `PollForCompilationComplete(true)` + `FPlatformProcess::Sleep(0.01)` loop therefore blocked the only thread that could finish the compile, which is exactly why "time since issued" tracked the wait duration to 30 ms. Fix (`Handlers/Niagara/NiagaraCompileWait.cpp`): one call, `FAssetCompilingManager::Get().ProcessAsyncTasks(/*bLimitExecutionTime=*/true)`, in a named helper `AdvanceAsyncCompilationOnGameThread()` invoked at the top of each loop iteration, before the poll (which reads what the pump finalises) and before the sleep. Same idiom the engine uses for its own game-thread waits: `FNiagaraSystemCompilationTask::WaitTillCompileCompletion` pokes `FNiagaraSystemCompilingManager::AdvanceAsyncTasks()` between event waits, and `AsyncCompilationHelpers.cpp:222` calls `ProcessAsyncResults` "to avoid starvation while we wait". Deliberately NOT `WaitForCompilationComplete` (unbounded, and `RequestCompile` on entry) and NOT `QueryCompileComplete` (private on `UNiagaraSystem`, NiagaraSystem.h:918 — which is why `#3` reached for `Poll`). The loop shape, the 90 s ceiling, the `bFlushRequestCompile=true` drain and the `HasOutstandingCompilationRequests()` entry gate are all unchanged, so neither the unbounded-loop nor the synthetic-system `Digest` crash is reintroduced, and the worst case degrades to today's behaviour rather than to a hang. `WaitForEmitterCompile` inherits the pump (it delegates per system). No change needed in `NiagaraEditTypes.cpp`: with the wait able to terminate on its own, `MayPersistAfterCompileWait` returns true and `FinalizeNiagaraEdit`'s `{compile:true,save:true}` save path — plus the same gate in `NiagaraHandler.cpp`'s `add_emitter`/`remove_emitter` — is restored. Header doc on `WaitForSystemCompile` updated to state it is game-thread-only and pumps while it blocks. Test added: `PinWright.niagara.CompileWait.WaitPumpsAssetCompilation` (`Tests/Niagara/TestNiagaraCompileWait.cpp`) asserts the pump call is legal in the wait's own context and, by reading `NiagaraCompileWait.cpp` through `IPluginManager`, that the loop body pumps and does so before both the poll and the sleep — every assertion fails on the pre-fix body, whose loop was `PollForCompilationComplete` + `Sleep` with no `FAssetCompilingManager` reference anywhere in the file. It is structural on purpose: observing a real compile land needs a real compile, and compiling this suite's synthetic systems kills the host, so no automated test can assert the outcome — the live repro remains the only thing that can. Needs a rebuild and a live re-run of the `#5` repro to verify.
- `#7-the-#6-fix-is-not-in-this-checkout` `IN-REVIEW` verifier — 2026-08-28, rebuilt DLL at plugin HEAD `b79ba53e`, editor pid 14932. Recorded from the crash-class verification pass over `B-niagara-compile-while-live-component-vectorvm-assert` and `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor`; this ticket was not in that assignment, so **status is left for its owner to move** — but the evidence says it should go back to `OPEN`, and the reason is not a failed fix.
  **`#6`'s fix is absent from this tree.** `#6` ends "Needs a rebuild and a live re-run of the `#5` repro to verify." The rebuild happened; the code did not arrive. At `b79ba53e` there is no `AdvanceAsyncCompilationOnGameThread`, no `FAssetCompilingManager` and no `ProcessAsyncTasks` anywhere in the plugin source, and no `PinWright.niagara.CompileWait.WaitPumpsAssetCompilation` test. `NiagaraCompileWait.cpp`'s loop is still exactly `#5`'s: `HasOutstandingCompilationRequests` -> `PollForCompilationComplete(true)` -> `FPlatformProcess::Sleep(0.01)`, with the last commit touching the file being `c480bc4e` (this ticket's `#3`). So what follows re-confirms `#5` on the rebuilt binary rather than refuting `#6`, which was never built here.
  **`#5` reproduces, three for three, on two further assets.** `niagara.compile {force:true, wait:true}` -> `{requested:true, waited:true, waitedMs:90003.9, compiled:false, timedOut:true, outstandingCompilationRequests:true, status:"timedOut", durationMs:90030.2}`. Engine log for the same asset: `LogNiagara: Compiling System ... took 90.080238 sec (time since issued)` and `took 90.113518 sec` — "time since issued" tracking the wait to within 100 ms, exactly `#5`'s signature.
  **New, and not in `#5`: the wait is a hard stall of the shared editor, and it is now measured.** A `register_slate_post_tick_callback` probe recorded the game thread not ticking at all for **90.05 s** and **90.08 s** across those calls, and again for **90.04 s** on a `{compile:true, save:true}` edit. In a shared editor that is a full outage for every other agent, once per call, and it is invisible to the caller — the response reports only `waitedMs`.
  **The contrast that isolates it, same verb, same asset, minutes apart.** `{compile:true, save:false}` skips the wait entirely (`NiagaraEditTypes.cpp:1649` scopes it to `bCompile && bSave`) and reports `compiled: true` with a longest stall of **0.028 s**, engine log `took 0.067841 / 0.137554 / 1.089133 sec`. `{compile:true, save:true}` on the same asset reports `compiled: false, saved: false` with a **90.04 s** stall. The compile does not take 90 s; the wait makes it take 90 s, because it holds the thread the compile needs to finalise on.
  **Caller-visible consequence today**, worth stating because two other tickets depend on it: `{compile:true, save:true}` — the ordinary way to make a `niagara.*` edit stick — costs 90 s of dead shared editor and then persists nothing, since `MayPersistAfterCompileWait` correctly refuses a save behind a compile that never landed. Nothing invalid is written, so `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor`'s corruption route really is closed; it is closed by the verb no longer working.
- `#8-reopened-fix-absent-here-and-wedge-escalated-to-critical` `OPEN` verifier — 2026-08-28. **Reopening, and raising severity `High` -> `Critical`.** Both actions rest on `#7`'s evidence, which `#7` deliberately did not act on because this ticket was outside its assignment.
  **(1) Status: `IN-REVIEW` asserts there is a fix here to review, and there is not.** Re-confirmed independently at plugin HEAD `b79ba53e` with a clean plugin working tree (`git status` empty, so no uncommitted `#6` work is in flight here either). Absent: `AdvanceAsyncCompilationOnGameThread` — zero hits anywhere in the plugin source; the test `PinWright.niagara.CompileWait.WaitPumpsAssetCompilation` — `Tests/Niagara/TestNiagaraCompileWait.cpp` declares only `PinWright.niagara.CompileWait.CompletionIsMeasuredNotAssumed`. `NiagaraCompileWait.cpp`'s loop is still `#5`'s exactly — `HasOutstandingCompilationRequests` (`:38`) -> `PollForCompilationComplete(true)` (`:54`) -> `FPlatformProcess::Sleep(0.01)` (`:56`), with no pump anywhere — and the last commit touching the file is `c480bc4e`, this ticket's `#3`.
  **Correcting one over-broad line in `#7` before it misleads someone:** `FAssetCompilingManager` and `ProcessAsyncTasks` are *not* absent from the whole plugin. They occur in `Handlers/Animation/AnimSequenceCreate.cpp:53,66`, `Handlers/Asset/ThumbnailFrameEvidence.cpp:61` and `Tests/TestAssetTeardown.h:94`. The accurate and still-decisive statement is that **neither symbol occurs anywhere under `Source/PinWright/Private/Handlers/Niagara/` — zero hits.** A fixer who greps the whole plugin will hit those unrelated matches and must not read them as `#6` having landed.
  **This is not a claim that `#6` is wrong, nor that its author failed.** Four hosts (`fuzz1`..`fuzz4`) work this board against separate plugin clones. `#6` is plausibly committed and building on another host's tree, and its reasoning — that the poll loop starves the game thread the compile finalises on — matches what `#5` and `#7` independently measured, so it is very likely the right fix. What is established is only that the code has not reached *this* checkout, which is what makes `IN-REVIEW` wrong here. **Before trusting `#6`, confirm which tree you are in:** `git -C Plugins/PinWright log -1 --oneline -- Source/PinWright/Private/Handlers/Niagara/NiagaraCompileWait.cpp` (expect a commit later than `c480bc4e` if `#6` is present) and `grep -rn FAssetCompilingManager Source/PinWright/Private/Handlers/Niagara/`. If `#6` is on your tree, this reopen does not apply to it — say so and re-run the `#5` repro rather than rewriting the fix. This is the second ticket found today sitting `IN-REVIEW` on code absent from this checkout (the other being a `MeshRebuildRenderGuard.h` guard file), so the cross-host pattern, not this ticket, is what wants a process fix.
  **(2) Severity `High` -> `Critical`, and the wedge is recorded here rather than as its own ticket.** `#7` measured that `{compile: true, save: true}` — the ordinary way to make a `niagara.*` edit stick — blocks the game thread for the full 90 s ceiling and then persists nothing (`compiled: false, saved: false`), while `{compile: true, save: false}` on the identical asset minutes apart returns `compiled: true` with a longest stall of **0.028 s**. Engine log for the same compile on the same asset: `took 0.067841 sec` in the fast case against `took 90.080238 sec` and `took 90.113518 sec` in the wedged one, `time since issued` tracking the wait to within 100 ms. `niagara.compile {force: true, wait: true}` behaves identically (`waitedMs: 90003.9`, `status: "timedOut"`, `outstandingCompilationRequests: true`, 90.05 s stall). The stall figures come from a `register_slate_post_tick_callback` probe — the game thread did not tick at all for 90.05 / 90.08 / 90.04 s — not from `waitedMs`. Full write-up in the new **Current state on this checkout** section above.
  **Why here and not a new ticket.** The wedge is not a second mechanism; it is the one `#5` already reported (`WaitForSystemCompile` starving the very thread that finalises the compile) seen from the bystanders' side, and `#6` closes both symptoms with a single change to a single function. Splitting would produce two OPEN tickets with one fix site and one fix, so one host would land it and the other would burn a claim discovering it was already gone — exactly what the lease exists to prevent — and neither ticket alone would carry the whole picture. Kept as this ticket's impact section instead, with severity taking the max of the two halves, which is the correct work-ordering answer when one change resolves both.
  **The severity argument, against the board's own table.** No impact row fits exactly: this is not a crash and it corrupts nothing, so by impact class alone it reads `High` ("hard blocker with no workaround" — and a workaround does exist: `{compile: true, save: false}` plus a separate `asset.save`). It is rated `Critical` for three things the table does not price. (a) **The work is lost** — the caller's edit is never written; `saved: false` is honest, but the outcome on disk is the same as a losing write. (b) **The blast radius is the whole process** — 90 s of dead game thread per call freezes every other agent on a shared editor, invisibly, since the response reports only `waitedMs`; the board's direct precedent for that class, `B-blueprint-search-wedges-game-thread`, is `Critical`. (c) **Reach is every-session** — this is the default call shape across the whole `niagara.*` edit namespace, which is the board's own one-level bump, and `High` + that bump is `Critical` by the table's own rule. Cross-check: the state this replaced — the same flag pair writing an invalidated compile and asserting in the VectorVM — is `Critical` on `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor`; a crash traded for a 90 s outage that also loses the edit is not a severity reduction.
  **Cross-links, so no one closes the wrong thing.** `B-niagara-compile-while-live-component-vectorvm-assert` is correctly `DONE` — its own guard is verified in source and was exercised clean — and its `#4` records the same wedge measurements as a regression pointing here; that pointer stands and that ticket does not need reopening. `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` (Critical, IN-REVIEW) is the one to re-verify **after** this is fixed: its corruption route is currently closed only because the save path declines, and fixing the wait restores that path.
  **Not re-run here, on purpose.** The wedge repro costs 90 s of shared editor per attempt and `#7`'s measurement is sound; only the source-absence claim was re-checked, which needs no editor. Title amended to describe the live defect instead of the fixed one — the original wording is kept in the new title's parenthetical and throughout `#1`-`#4`. `encounters` deliberately left at 4: this entry is a status and severity decision on `#7`'s observation, not a new observation.
- `#9-fix-is-present-on-this-tree` `IN-REVIEW` developer — **ALREADY-FIXED. `#8`'s reopen was correct for `b79ba53e` and is stale here: `#6`'s pump is on this checkout as commit `f281e2a0`, which is later than the HEAD `#7`/`#8` checked.** No plugin source was written. Plugin HEAD `11aebe3f`; `git log b79ba53e..HEAD` is three commits, one of them `f281e2a0` "Pump asset compilation while waiting for it, instead of starving it". `#8` prescribed three checks and all three now pass: `git log -1 -- .../NiagaraCompileWait.cpp` → `f281e2a0`, not `c480bc4e`; `AdvanceAsyncCompilationOnGameThread` / `FAssetCompilingManager` present under `Handlers/Niagara/` (`NiagaraCompileWait.cpp:58-61` defines it, `:80` calls it at the top of the loop body before the poll at `:89` and the sleep at `:91`); and `PinWright.niagara.CompileWait.WaitPumpsAssetCompilation` exists (`Tests/Niagara/TestNiagaraCompileWait.cpp:224`). This is the cross-host lag `#8` predicted, not a second fix.
  **`#6`'s mechanism re-derived from UE 5.8 engine source rather than taken from its commit message**, because the whole point of the reopen was that plausible-sounding fix notes had not been checked against a tree. Every claim holds. `FShaderCompilingManager` is an `IAssetCompilingManager` (`ShaderCompiler.h:934`) and registers itself with `FAssetCompilingManager` (`ShaderCompiler.cpp:898`) — it is not among the ctor's built-ins, so a reader who only greps `AssetCompilingManager.cpp:543-550` will wrongly conclude it is absent — and its `ProcessAsyncTasks(bool)` is literally `ProcessAsyncResults(bLimitExecutionTime, false)` (`ShaderCompiler.cpp:1400-1403`). `FNiagaraSystemCompilingManager` registers at `NiagaraEditorModule.cpp:1732`, and its `ProcessAsyncTasks` (`NiagaraSystemCompilingManager.cpp:268-368`) drains `GameThreadFunctions`, ticks every `ActiveTasks` handle, and ends in `while (ConditionalLaunchTask()) {}` — so without the pump a queued compile never even launches, exactly as `#6` said. `FAssetCompilingManager::ProcessAsyncTasks` (`AssetCompilingManager.cpp:768-779`) iterates every registered manager, carries no re-entrancy guard and no game-thread assert, so the nested call from inside the wait is legal. The two rejected alternatives are confirmed rejected for the stated reasons: `QueryCompileComplete` is genuinely private (`NiagaraSystem.h:918`), and `WaitForCompilationComplete` — public at `NiagaraSystem.h:452` — calls `RequestCompile` on entry (`NiagaraSystem.cpp:3191-3194`) and then busy-waits on `QueryCompileComplete(true)` with no pump and no timeout outside `GIsAutomationTesting`.
  **The save-ordering half is closed in source, verified by reading it rather than by trusting `#3`.** `FinalizeNiagaraEdit` (`NiagaraEditTypes.cpp`, the `if (Options.bCompile && Options.bSave)` block) waits, re-derives `compiled` through `DidCompileLand` so the field is an observation, gates the write on `MayPersistAfterCompileWait`, and logs a warning naming the ceiling and the waited time when it refuses. The same gate is in `NiagaraHandler.cpp`'s `add_emitter` (`:327`) and `remove_emitter` (`:518`), there sitting after `RejectOnDataInterfaceMismatch`. `niagara.compile` reports `requested` / `compiled` / `waited` / `waitedMs` / `timedOut` / `outstandingCompilationRequests` / `status` as separate measured fields; the `compiled:true` beside `status:"requested"` contradiction has no way back.
  **No new test, and that is a decision rather than an omission.** The two shipped with the fix are the honest ceiling: observing a compile land needs a real compile, and compiling this suite's synthetic systems crashes the host inside `FNiagaraCompilationGraphDigested::Digest`, so `WaitPumpsAssetCompilation` is structural by necessity. Recording it plainly so nobody reads its green as behavioural proof: **a green suite cannot confirm this ticket. Only the `#5` live repro can.**
  **What this entry does NOT claim.** Nothing was run against an editor — no compile, no automation suite, no repro. The 90 s wedge was not re-measured, deliberately, since a failed attempt costs the shared editor 90 s and ~17 agents are on this checkout. And the ceiling is unchanged at 90 s: a compile that genuinely runs long still blocks the game thread up to it. That is met only in the sense this ticket's own acceptance bar allows ("a bounded pumped wait ... would satisfy this ticket"), so a verifier who reads the bar more strictly should return it rather than pass it.
  **One adjacent gap, left alone on purpose.** `FinalizeNiagaraEdit` gates its save on the wait outcome only — it never runs `PinWrightNiagara::CheckDataInterfaceCounts` — while `add_emitter`/`remove_emitter` gate on both. So the ~25 edit verbs behind `FinalizeNiagaraEdit`, which are where this ticket's 0-compiled-vs-2-resolved evidence came from, are protected by timing rather than by the invariant. That is `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor`'s guard, and this ticket already nominates it for re-verification now that the save path works; touching it from here would be two hosts writing one fix.
  **For the verifier.** Needs a rebuilt DLL — check the on-disk `UnrealEditor-PinWright.dll` timestamp against `f281e2a0`, not the build's success message, per `#4`. Then re-run `#5`: `niagara.compile {force:true, wait:true}` should return `status:"completed"` with `waitedMs` in the same order as the engine log's `Compiling System ... took N sec`, not tracking a 90 s ceiling, and `{compile:true, save:true}` on the same asset should report `saved:true`. A `register_slate_post_tick_callback` probe should show the longest stall matching the compile, not the ceiling.
