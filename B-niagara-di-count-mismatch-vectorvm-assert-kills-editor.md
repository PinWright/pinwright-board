---
id: B-niagara-di-count-mismatch-vectorvm-assert-kills-editor
title: "Editor crash: a Niagara system left with 0 compiled DataInterfaceInfos against 2 resolved ones asserts in VectorVM on its NEXT TICK, so any verb that forces a re-tick (sequencer.set_playhead opening the Level Sequence editor) kills the shared editor minutes after the write that broke it"
status: IN-REVIEW
severity: Critical
category: bug
tags: [niagara, vectorvm, data-interface, compile, presave, sequencer, set-playhead, editor-kill, delayed-fault, shared-editor, latent-corruption]
encounters: 1
lastSeen: 2026-08-27T19:41:41+05:00
---

# A data-interface count mismatch is logged as a Warning and detonates later as an `appError`

`/Game/Atlantis/VFX/NS_FishSchool` is in a state where its compiled script carries **zero**
`DataInterfaceInfos` while the resolved set carries **two**. Every tick of that emitter then asks
`VectorVM` for a data set index the exec context does not have, and the engine asserts on a
background worker — which is an `appError`, so the whole editor dies.

The two events are ~4.5 minutes and an unrelated agent apart, which is what makes this expensive:
nothing in the crash names the write that caused it, and nothing in the write's own response says
anything went wrong.

## The write (14:37:01 UTC), logged as a Warning and otherwise silent

`Saved/Logs/EAContentExamples58.log:4421-4425`:

```
LogNiagara: Warning: Data interface count mismatch during script presave.
  Invaliding compile results (see full log for details).
  Script: /Game/Atlantis/VFX/NS_FishSchool.NS_FishSchool:Fountain.UpdateScript
LogNiagara: Compiled DataInterfaceInfos:                      <-- empty
LogNiagara: Resolved DataInterfaceInfos:
LogNiagara:   Name:Emitter.Scale Alpha.FloatCurve, Type: NiagaraDataInterfaceCurve
LogNiagara:   Name:Emitter.VectorField32,          Type: NiagaraDataInterfaceVectorField
```

`NS_FishSchool` is one of the systems the host spec's *"Ruling: Niagara emitters may be duplicated
from stock templates"* covers — the emitter is still named `Fountain`, i.e. a duplicated stock
template rewritten in place. The two orphan data interfaces (`Scale Alpha.FloatCurve`,
`VectorField32`) are template leftovers the rewrite did not remove; the recompile dropped them from
the compiled list and left them in the resolved list.

"Invaliding compile results" is the engine's own mitigation and it is **not enough**: the system
keeps ticking with the mismatched context.

## The detonation (14:41:41 UTC), from an unrelated verb

`Saved/Logs/EAContentExamples58.log:4644-4676`. My call was
`sequencer.set_playhead {path:"/Game/Atlantis/Cine/LS_Atlantis_Flythrough", frame:0}` — a
read-shaped scrub. It returned success. 0.575 s later the editor was gone:

```
14.41.41:045  LogAssetEditorSubsystem: Opening Asset editor for LevelSequence /Game/Atlantis/Cine/LS_Atlantis_Flythrough
14.41.41:263  LogSequenceNavigator: Tool instance 'LevelSequenceEditor' provider 'LevelSequence' registered
14.41.41:838  LogWindows: Error: appError called: Assertion failed: DataSetIdx < ExecCtx->DataSets.Num()
              [VectorVMRuntime.cpp:421]
```

Callstack, innermost first — note it is a **concurrent** Niagara tick on `Background Worker #3`,
not the game thread:

```
VectorVM::Runtime::SetupBatchStatePtrs()              VectorVMRuntime.cpp:421
VectorVM::Runtime::ExecVectorVMState()                VectorVMRuntime.cpp:2286
FNiagaraScriptExecutionContextBase::ExecuteInternal() NiagaraScriptExecutionContext.cpp:262
FNiagaraEmitterInstanceImpl::Tick()                   NiagaraEmitterInstanceImpl.cpp:1735
FNiagaraSystemInstance::Tick_Concurrent()             NiagaraSystemInstance.cpp:2724
FNiagaraSystemSimulation::Tick_Concurrent()           NiagaraSystemSimulation.cpp:1921
```

`DataSetIdx < ExecCtx->DataSets.Num()` is the direct consequence of the 0-vs-2 mismatch: the
bytecode indexes a data set the exec context never allocated.

## Why `sequencer.set_playhead` is the trigger, and why that matters for this plugin

`set_playhead` **opens the Level Sequence in the Sequencer editor** as a documented side effect
(`sequencer.set_playhead.md`: *"`set_playhead` therefore opens the named sequence when it is not
the open one"*). Opening it registers the sequence tool and forces a full re-evaluation of the
level — including putting bound Niagara components into age-addressable mode, which is precisely
the behaviour the wiki advertises as the reason to use this verb (`sequencer.md` § *Driving the
editor viewport without PIE*). That re-evaluation was the first tick `NS_FishSchool` got after
being corrupted, and it was fatal.

So the verb the plugin documents as **the** deterministic instrument for frame bursts is also the
verb most likely to detonate a latently-corrupt Niagara system, and there is no crash-safe variant:
`open:false` only makes it refuse with `SEQUENCE_NOT_OPEN`, it does not give you a scrub that
avoids the asset editor.

## Suggested fix, in priority order

1. **Make the mismatch a hard error at the write, not a Warning.** Whatever `niagara.*` verb wrote
   `NS_FishSchool` last (an edit followed by a save) should surface `LogNiagara`'s *"Data interface
   count mismatch during script presave"* as a failed response — the asset is corrupt from that
   moment and the caller is the only one who can still fix it cheaply. Compare compiled vs resolved
   `DataInterfaceInfos` counts after compile and refuse to report success when they differ. Same
   shape as the existing rule *"Never return bare `compiled: true` from anything IR/graph-shaped"*
   (`agent-conventions.md` § Error conventions) — Niagara needs it too.
2. **Add a pre-flight to `sequencer.set_playhead`** (and to any verb that opens an asset editor or
   forces a level re-evaluation): refuse, with a named system, when a Niagara system in the level
   has a compiled/resolved data-interface count mismatch. A typed `NIAGARA_SYSTEM_CORRUPT` naming
   `NS_FishSchool` costs one call; the alternative costs every agent in the editor everything they
   have not saved.
3. **`niagara.validate` should catch this.** The host spec records that `niagara.validate` strict
   reports zero errors on assets that provably do not work (§ *Ruling: Niagara emitters may be
   duplicated from stock templates*, defect D27). This is a second, sharper instance: a system that
   is not merely inert but **fatal on tick**, and validate is green. A DI-count check is a cheap,
   exact test — no heuristics needed.
4. Independently: the duplicate-from-stock-template workflow the host spec mandates leaves orphan
   template data interfaces (here `Scale Alpha.FloatCurve` and `VectorField32` from the stock
   `Fountain` emitter). A `niagara.*` verb to enumerate and remove resolved-but-uncompiled data
   interfaces would close the hole that ruling opens.

## Impact

Critical, and worse than its severity suggests because it is a **delayed** fault. The corrupting
write returns success and logs a Warning; the editor survives for minutes; then an unrelated agent
running an unrelated read-shaped verb takes the whole shared editor down and gets blamed for it.
Five editor crashes were recorded in this session and the crash log is the only place the causal
chain is visible.

Concretely on this host: it cost the Sequencer frame-burst verification route entirely. The brief
for that work names `sequencer.set_playhead` as the instrument *because* it evaluates bound Niagara
at the addressed age; with `NS_FishSchool` in this state, the instrument is a kill switch. The
fallback — `sequencer.get_binding_transform` to interrogate the evaluated pose, then
`render.capture_open_level` with that pose passed explicitly — never opens the asset editor and
does not crash, but it gives up exactly the Niagara-at-age evaluation that made `set_playhead`
worth using.

## Distinct from related tickets

- `B-model-compile-live-niagara-mesh-renderer-raytracing-assert` (filed today; the duplicate
  `B-static-mesh-rebuild-crashes-live-niagara-mesh-renderer` was merged into it and deleted) is a
  **renderer** fault — a live Niagara *mesh renderer* holding a stale LOD index, asserting on the
  render thread / in the ray-tracing gather. This one is the **simulation** side: VectorVM on a concurrent worker,
  no mesh involved, triggered by a tick rather than by a mesh rebuild.
- `B-niagara-authored-emitter-forces-inert` is the *inert* failure (no `NiagaraNodeEmitter`, so the
  emitter never runs). This is the opposite: the emitter does run, and running is what kills the
  process.
- `B-niagara-create-node-unfinalized-graph-node-creator-fatal` is a synchronous
  return-path/destructor fault on the game thread from one bad argument. This one is asynchronous,
  latent, and triggered by a different caller than the one that caused it.

## Environment

UE 5.8, `EAContentExamples58`, `/Game/Maps/Atlantis`, 2026-08-27T19:41:41+05:00 (log UTC 14:41:41;
logs are UTC+0, machine UTC+5). Log: `Saved/Logs/EAContentExamples58.log:4421-4425` (the write) and
`:4644-4676` (the crash). Offending asset: `/Game/Atlantis/VFX/NS_FishSchool`, emitter `Fountain`.

## History

- `#1-observed` `OPEN` reporter — Found as the Sequencer/cine agent, from the crash log rather than
  deliberately: my `sequencer.set_playhead` was the trigger, another agent's Niagara edit ~4.5 min
  earlier was the cause. Not re-run to confirm — reproducing it costs every agent in the shared
  editor another crash, and the log chain (Warning at `:4421`, assert at `:4666`) is unambiguous
  without it. Two other agents had already deleted three `NiagaraActor`s at 14:39:22 (`:4604-4606`),
  which suggests the systems were suspected before the crash; the level was not saved, so the
  actors came back on restart and the trap was re-armed.
- `#2-write-path-di-count-gate` `IN-REVIEW` developer — Added a post-write data-interface validity
  gate to the two system-mutating verbs in `Source/PinWright/Private/Handlers/Niagara/NiagaraHandler.cpp`
  (`niagara.add_emitter`, `niagara.remove_emitter`). New
  `Handlers/Niagara/NiagaraDataInterfaceConsistency.{h,cpp}` runs the same comparison the engine
  runs in `FNiagaraScriptRuntimeCompiledData::ValidateWithScript` — compiled
  `FNiagaraVMExecutableData::DataInterfaceInfo.Num()` vs `ResolvedDataInterfaces.Num()`, per script,
  via `ForEachScriptWithOwningContext` — and returns Consistent / Mismatched / Unverified. Scripts
  with no resolved entry (an emitter handle added since the last compile) are skipped rather than
  flagged, so the gate has no false-positive direction. On Mismatched the handlers now refuse:
  no save, `NIAGARA_DATA_INTERFACE_MISMATCH` naming each offending script with both counts, and the
  in-memory mutation facts echoed on the error payload. On a pass the verdict is echoed as
  `dataInterfaceCheck` so an unverified pass is not reported as a verified one. Test
  `PinWright.niagara.data_interface_consistency.WritePathReportsVerdict`
  (`Source/PinWright/Private/Tests/Niagara/TestNiagaraDataInterfaceConsistency.cpp`) asserts the
  guard, not the crash. Suggested fixes 2 (`sequencer.set_playhead` pre-flight), 3
  (`niagara.validate` DI-count check) and 4 (a verb to enumerate/remove orphan resolved DIs) are
  not addressed here — they are outside this file and want their own tickets.
- `#3-root-cause-confirmed-compile-save-race` `IN-REVIEW` developer — Independent confirmation of the
  mechanism, from a log-forensics pass in the `EAContentExamples58` session. **The root cause is
  passing `compile: true` and `save: true` on the same `niagara.*` call.** The compile is
  asynchronous, so the save writes an invalidated compile — the `0` compiled versus `2` resolved
  `DataInterfaceInfo` state this ticket describes. Separating them into `niagara.compile {force,
  wait}` followed by `asset.save` took one system from **44 mismatch warnings to 0**, which is a
  measurement rather than an inference. This was flagged as an unproven hypothesis in
  `#2-write-path-di-count-gate` ("saving while a compile is in flight is a plausible route into the
  mismatch, but I could not prove it"); it is now proven, by a different session, from the log.
  Consequence for the fix: the post-write gate shipped in `#2` is a **backstop, not the cure** — it
  refuses to persist a corrupt system, which is what stops the editor kill, but the underlying race
  is only closed when `compile` actually completes before `save` runs. That is
  `B-niagara-compile-wait-does-not-wait`, whose `wait: true` is currently ignored; the two tickets
  should be verified together, and this one should not be marked `DONE` on the gate alone.
  **This ticket is deliberately NOT merged into
  `B-niagara-compile-while-live-component-vectorvm-assert`.** The two describe the same crash *event*
  but are different defects with different fix sites — that one is a missing `KillSystemInstances`
  before `RequestCompile`, this one is a compile/save ordering race.
