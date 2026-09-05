---
id: B-niagara-di-count-mismatch-vectorvm-assert-kills-editor
title: "Editor crash: a Niagara system left with 0 compiled DataInterfaceInfos against 2 resolved ones asserts in VectorVM on its NEXT TICK, so any verb that forces a re-tick (sequencer.set_playhead opening the Level Sequence editor) kills the shared editor minutes after the write that broke it"
status: IN-REVIEW
severity: Critical
category: bug
tags: [niagara, vectorvm, data-interface, compile, presave, autosave, sequencer, set-playhead, editor-kill, game-thread-hang, delayed-fault, shared-editor, latent-corruption]
encounters: 4
lastSeen: 2026-09-05T22:37:00+03:00
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
- `#4-autosave-arms-it-with-no-plugin-save` `OPEN` reporter — **New route into the same 0-vs-2 state,
  observed twice in one session, with no PinWright save verb involved at all.** A `niagara.*` edit
  passed `compile:false, save:false` still (a) marks the package dirty and (b) leaves the compiled
  scripts stale. The **editor's own 10-minute autosave** then presaves that dirty package, hits the
  mismatch, and logs `Invaliding compile results`. Mitigation `#3` ("separate compile from save")
  cannot help here, because there is no save to separate — the caller did not ask for one, and the
  arming window is 0-10 minutes wide after *any* uncompiled `niagara.*` edit.

  **Occurrence A — wedged the shared editor for 14 minutes.** 21:33:55+05 I called
  `niagara.set_static_switch {assetPath:"/Game/Atlantis/VFX/NS_FishSchool", inputName:"Mass Mode",
  value:2, compile:false, save:false}` purely to read back the resolved enumerator (the oracle this
  board recommends for `B-niagara-static-switch-enum-value-map-undiscoverable`). It returned
  `{"value":"NewEnumerator3"}` and nothing else. At 21:37:56+05 the autosave fired
  (`Saved/Logs/EAContentExamples58-backup-2026.08.27-16.51.23.log:3127-3144`): `SAVEPACKAGE ... 
  NS_FishSchool_Auto2.uasset ... AUTOSAVING=true`, then the mismatch on **both** SpawnScript and
  UpdateScript, 0 compiled against the same two resolved DIs this ticket names. **The game thread
  never completed another tick.** `EDITOR_NOT_READY` climbed 133 s -> 683 s; the last non-audio log
  line for the next 14 minutes was that autosave; the editor had to be killed. Three live
  `VFX_FishSchool_*` components of that system were in the level at the time. Note the shape
  difference from `#1`: not the VectorVM `appError` but a silent permanent game-thread hang, so
  there is no callstack and no crash log — only the Warning.

  **Occurrence B — identical write, no wedge, one variable changed.** 22:00+05, five
  `niagara.set_module_input {compile:false, save:false}` calls on the same system; autosave at
  22:03:02+05 logged the identical mismatch pair (`Saved/Logs/EAContentExamples58.log:3048-3059`).
  The editor survived. The **only** difference from A: beforehand I had detached all three
  components with `UNiagaraComponent::SetAsset(nullptr)`, so nothing was ticking the invalidated
  scripts. That is a controlled-ish confirmation that the live component is the detonator and the
  presave is the primer, and it is why this ticket and
  `B-niagara-compile-while-live-component-vectorvm-assert` need fixing together.

  **`effect.deactivate_niagara` is not a sufficient guard.** Its handler
  (`Handlers/VFX/EffectHandler.cpp:716`) calls `UNiagaraComponent::Deactivate()`, which stops
  spawning but leaves existing particles simulating for their full lifetime — 26-42 s on these
  emitters — and leaves `IsActive()` reading `true`. `DeactivateImmediate()` is not exposed to
  Python and has no RPC. The only reliable detach available to a caller today is
  `SetAsset(nullptr)` via `python.execute`, which destroys the system instance outright.

  **Repair recipe, verified against disk.** With no component referencing the system:
  `niagara.compile {force:true, wait:true}` as its own call, then `asset.save {force:true}` as its
  own call. Presave then logs no mismatch and the `.uasset` mtime advances. Applied to all three of
  `NS_FishSchool` / `_Orange` / `_Silver`; zero mismatch warnings across six subsequent saves.

  **Suggested fix, in addition to the four above.** A graph-mutating `niagara.*` verb called with
  `compile:false, save:false` currently hands the caller a dirty package with stale compiled scripts
  and says nothing. It should do one of: (a) refuse when the target system has live components in
  the level, naming them — this also closes the sibling ticket; (b) warn in the response that the
  asset is now autosave-armed until a compile lands, so the caller knows the clock is running;
  or (c) not leave the package dirty when neither compile nor save was requested. (a) is the
  strongest: it is the only one that protects the *other* agents in a shared editor, who never see
  this caller's response.

- `#5-guard-test-is-inert-on-this-host` `IN-REVIEW` verifier — 2026-08-28. Plugin rebuilt at `b79ba53e` and the scoped suite run against it. **The new test that guards this ticket reports success without running a single assertion on this host.** `PinWright.niagara.data_interface_consistency.WritePathReportsVerdict` completed `Result={Success}` and emitted `PINWRIGHT_ASSERTIONS_SKIPPED ... reason=niagara-resolved-di-unavailable -- no resolved data-interface set on '/Niagara/DefaultAssets/Templates/Systems/SimpleExplosion.SimpleExplosion' in this host/engine build`. So the fix for a Critical editor-killing assert is, on this machine, covered by a test that cannot fail. Credit where due: the plugin's own `TestSkipReporting.h` surfaced this honestly rather than letting it pass as a real green — `check_suite_log.py` classified the whole run `COMPLETED_WITH_SKIPS` and named the test. The handler-side change was not otherwise exercised in this pass: the `dataInterfaceCheck` field WAS observed live, returning `"consistent"` from `niagara.add_emitter` on a fresh scratch system, which proves the check runs and reports but not that it refuses a genuine mismatch. Needs a fixture that does not depend on an engine template's resolved DI set — otherwise this ticket's guard is unverifiable here and will stay that way.

- `#6-gate-runs-refusal-still-unexercised-and-the-cure-regressed-the-save` `IN-REVIEW` verifier — 2026-08-28, rebuilt DLL at plugin HEAD `b79ba53e`, editor pid 14932. **Left IN-REVIEW deliberately, not for lack of effort: the thing this ticket needs decided could not be produced on this host.** Stating that explicitly rather than guessing a verdict.
  **Measured — the gate is real and runs.** `dataInterfaceCheck` came back on every `niagara.add_emitter` and `niagara.remove_emitter` call in this pass (six, across two systems, including a stock `Fountain` template emitter added to a fresh system — the same emitter and the same two orphan data interfaces this ticket names). Every verdict was `"consistent"`. Source confirms the comparison is the engine's own: `NiagaraDataInterfaceConsistency.cpp:84-85` compares compiled `FNiagaraVMExecutableData::DataInterfaceInfo.Num()` against `ResolvedDataInterfaces.Num()` per script via `ForEachScriptWithOwningContext`, the three-way verdict exists (`.h:26-36`, `Unverified` when no script could be compared or the resolved header is unavailable), and the refusal branch at `NiagaraHandler.cpp:97` sends `NIAGARA_DATA_INTERFACE_MISMATCH` with a `mismatchedScripts` array and returns before the save at `:329` / `:520`.
  **Not measured — no mismatch could be manufactured, so the refusal direction was never taken.** Across roughly thirty Niagara operations, including the exact `{compile:true, save:true}` combination `#3` names as the root cause and an uncompiled structural edit persisted with `save:true`, the session log shows **0** `Data interface count mismatch during script presave` lines. `#5`'s finding therefore stands unchanged and is now confirmed from the other direction: the guard for a Critical editor-killing assert has no exercised coverage on this host, neither in the suite (where it skips) nor by hand (where the state cannot be reached). Reporting `"consistent"` proves the check runs; it does not prove it refuses.
  **Three findings that narrow what `#2` and `#3` claim.**
  (a) **Scope is two verbs, not the write path.** `RejectOnDataInterfaceMismatch` / `CheckDataInterfaceCounts` are called from `add_emitter` and `remove_emitter` only — a grep over `Private/Handlers` finds them nowhere else. `FinalizeNiagaraEdit`, `niagara.set_property`, `niagara.set_module_input`, `niagara.set_static_switch` and `niagara.compile` have no DI gate. `#4`'s occurrence A was a `set_static_switch`, so the route that wedged the editor for 14 minutes is still ungated.
  (b) **`#4`'s autosave route is untouched and was reproduced as a state, not as a crash.** `niagara.add_emitter {compile:false, save:true}` returned `saved: true` with `dataInterfaceCheck: "consistent"`, persisting a package whose compiled scripts are stale — precisely the primer `#4` describes, with nothing in the response telling the caller the autosave clock is running. `#4`'s suggested fix (a) / (b) / (c) remains undone.
  (c) **`#3`'s cure now fails in a new way, and `#3` was right that the two tickets must be verified together.** The compile/save race IS closed — the save is refused when the compile has not landed, so nothing invalid is written. But the wait it depends on never lands: `{compile:true, save:true}` wedges the game thread for the full 90 s ceiling and returns `compiled:false, saved:false`, while the identical edit with `save:false` compiles in 0.07-1.09 s with a 0.028 s stall. So the protection works by never persisting anything, at a cost of 90 s of dead shared editor per call. **The cause is that `B-niagara-compile-wait-does-not-wait` `#6`'s fix is absent from this checkout** — no `AdvanceAsyncCompilationOnGameThread`, no `FAssetCompilingManager`, no `ProcessAsyncTasks` anywhere in the plugin source at `b79ba53e`. Measurements in that ticket and in `B-niagara-compile-while-live-component-vectorvm-assert` `#4`.
  **What would close this ticket:** a fixture that puts a system into the 0-compiled / 2-resolved state deliberately, so the refusal branch can be taken once, live. `#5` asked for the same thing for the test; the same fixture serves both. Until it exists the guard is unfalsifiable here and the ticket should not be marked DONE.

- `#7-the-missing-fixture-exists-on-disk-in-this-checkout` `IN-REVIEW` reporter — plugin rebuilt
  2026-09-05, editor gateway port 27145, UE 5.8, `EAContentExamples58`. **`#5` and `#6` both close
  with "needs a fixture that puts a system into the 0-compiled / N-resolved state". One is sitting
  in this checkout, produced by ordinary use, not by a test.**
  **The system.** `/Game/FPS/VFX/NS_Blood`, four emitters (`Drips` / `Spray` / `Mist` / `Burst`),
  last written by a previous agent's pass. Every one of twelve `niagara.set_module_input` calls I
  made against it before compiling returned `success: true` **and** `dataInterfaceCheck:
  "mismatched"`, with `mismatchedScripts` naming concrete counts:
  `NS_Blood:Drips.SpawnScript` and `.UpdateScript` at `compiledDataInterfaces: 0` /
  `resolvedDataInterfaces: 3`; `Spray.SpawnScript`/`.UpdateScript` and `Mist.SpawnScript`/
  `.UpdateScript` at `0` / `2`. That is this ticket's state exactly, on a real authored asset.
  **It is not a load artifact.** `niagara.validate {level:"basic"}` on the sibling
  `/Game/FPS/VFX/NS_Impact_Concrete` — same folder, same tooling, saved three days earlier, also
  never compiled in this session — returns `dataInterfaceCheck: "consistent"`. Same session, same
  uncompiled-since-load condition, opposite verdict, so the check discriminates and `NS_Blood` was
  genuinely persisted primed.
  **Narrows `#6`(a).** `#6` measured the gate as reachable from `add_emitter` / `remove_emitter`
  only, with `set_module_input` ungated. In the 2026-09-05 build `set_module_input` **does** carry
  `dataInterfaceCheck` and did emit `mismatched` on every call — so the reporting has been widened
  to the write path since `b79ba53e`. It reports and does not refuse: all twelve edits were applied
  and the package left dirty. Whether the *save* refusal fires from this route is still unmeasured
  here, because I compiled before saving and never presented a mismatched system to `asset.save`.
  **The repair in `#4` holds, third confirmation.** `niagara.compile {force:true, wait:true}` as its
  own call returned `status: "completed", compiled: true, timedOut: false, waitedMs: 3968`; then
  `asset.save {force:true}` returned `saveState: "written"`, `.uasset` mtime advanced
  2026-09-05 20:55 -> 22:21:30 +03:00 and size 928774 -> 929928 B; `niagara.validate {level:
  "strict"}` then reports `dataInterfaceCheck: "consistent"`, `valid: true`, zero errors, and zero
  `NCS_Error` scripts (10 `NCS_UpToDate`, 8 `null` on the EmitterSpawn/EmitterUpdate pair).
  **Why this is worth the Critical.** The VFX coordinator on this package reports three editor kills
  attributed to captures. A capture spawns the system and ticks it — the detonator this ticket
  names — and `NS_Blood` was on disk in the primed state, in the set of thirteen systems the next
  capture pass sweeps. The delayed, unattributable shape `#1` describes is exactly how those kills
  presented.
  **What this gives the fix.** A reproducible mismatched asset needs no manufacturing: check out
  `NS_Blood` at its pre-`22:21` revision and the refusal branch at `NiagaraHandler.cpp:97` can be
  taken live, once, which is the single measurement `#5` and `#6` both say is missing.
