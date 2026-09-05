---
id: B-niagara-validate-green-while-scripts-ncs-error
title: "niagara.validate reports valid:true with zero errors while compile.valid is false and particle scripts are NCS_Error — the system then refuses to activate"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, validate, false-success, silent-noop, compile-state, NCS_Error, strict, activation, no-readback]
encounters: 3
lastSeen: 2026-09-02
---

# A system whose scripts failed to compile validates clean at `level: strict`

`niagara.validate` publishes a nested `compile` block that carries a `valid` boolean and a
per-script `compileStatus`. When a script's status is **`NCS_Error`** — a real, populated compile
failure, not the `null` / `NCS_Unknown` "not compiled yet" state that
`B-asset-dump-niagara-compile-state-stale` and `E-niagara-validate-compile-state-uninitialized-undocumented`
already cover — the failure is **not propagated to the top-level verdict**. `valid` stays `true`,
`errors` stays empty, and the only warning raised is the benign `COMPILE_STATE_UNINITIALIZED`.

Measured 2026-09-02 on the FPS VFX package, editor gateway port 27145, UE 5.8.

## What I called

    call("niagara.validate", {assetPath: "/Game/FPS/VFX/NS_Explosion",     level: "strict"})
    call("niagara.validate", {assetPath: "/Game/FPS/VFX/NS_Smoke_Grenade", level: "strict"})

…plus the other twelve systems in the same package as controls.

## What happened

| system | top-level `valid` | top-level `errors` | `compile.valid` | scripts at `NCS_Error` |
|---|---|---|---|---|
| `NS_Explosion` | **`true`** | **`[]`** | `false` | 4 — `Shockwave.SpawnScript`, `Shockwave.UpdateScript`, `DustRing.SpawnScript`, `DustRing.UpdateScript` |
| `NS_Smoke_Grenade` | **`true`** | **`[]`** | `false` | 6 — `Smoke`, `Core`, `Wisps` × Spawn+Update |
| the other 12 systems | `true` | `[]` | `true` | 0 |

`dataInterfaceCheck` was `consistent` on all fourteen, `hasOutstandingCompilationRequests` and
`hasActiveCompilations` were both `false`, so this is not a compile still in flight.

**The two systems the verb green-lit are the two that do not run.** In the same session, in
`/Game/FPS/Test/T_VFX`:

    call("effect.spawn_niagara",    {systemPath: "/Game/FPS/VFX/NS_Explosion", ...})   -> ok
    call("effect.activate_niagara", {systemName: "VC_Explosion", reset: true})
      -> {"requestedActive": true, "active": false, "activationWarning": "... IsActive() reads
          false afterwards, so NOTHING is running ..."}

Identical result for `NS_Smoke_Grenade`. The other twelve systems, spawned and activated the same
way in the same world in the same minute, all returned `active: true`. So the correlation is exact:
`compile.valid:false` / `NCS_Error` ⇒ the component refuses to activate ⇒ zero pixels — and the
verb whose whole job is to answer "is this asset healthy" answers yes.

## What I expected

At least one top-level `error` naming each `NCS_Error` script, and `valid: false`. A script that
failed to compile is not a severity question the `level` parameter should be able to soften: like
`EMITTER_NOT_IN_SYSTEM_GRAPH` and `NIAGARA_DATA_INTERFACE_MISMATCH`, a system that cannot run is
not acceptable under any reading of the asset, and `basic` should raise it too.

## Which wiki page misled me

`Saved/PinWright/wiki/niagara.validate.md` states: *"the top-level `errors` / `warnings` /
`issues` arrays apply this `level` normalization and are **authoritative**. The nested
`compile.issues` block carries the raw pre-normalization dumper severity"*. That sentence tells a
caller to trust the top level and ignore the nested block. Here the nested block is the **only**
place the failure appears, and even there it appears as `compile.valid: false` and a per-script
`compileStatus`, never as an issue — so a caller who follows the documented rule cannot see it at
all. The page's compile-state section only discusses `null` / `NCS_Unknown`; `NCS_Error` is not
mentioned anywhere on it.

## Cost

The VFX builder's report (`Docs/fps/reports/vfx-build-01.md`) records "All 14 `valid:true`, zero
errors, `dataInterfaceCheck:"consistent"`" as its structural verification. That statement is a
faithful quote of the verb and it is wrong about the assets: two of the fourteen — the explosion
and the smoke grenade, two of the six items the stream's quality bar names explicitly — were dead
on disk and shipped as verified. Nothing else in the toolchain disagreed.

## Root-cause guess

Not read from source. The shape of the bug is that the compile-state reader already distinguishes
`NCS_Error` from `NCS_Unknown` (it prints the former literally and maps the latter to `null`, per
`niagara.compile-state`), and `compile.valid` already goes false for it — so the missing step is
only the promotion of that state into the issues array that the top-level normalizer consumes.

## Workaround

Do not read `valid`. Read the nested block and fail on it yourself:

    compile.valid === false
      || compile.scripts.some(s => s.compileStatus === "NCS_Error")

and, for anything that must actually render, confirm with `effect.activate_niagara` and require
`active: true` — that verb *is* honest, and its `activationWarning` was the first signal in this
session that disagreed with everything else.

## History
- `#1-filed` `OPEN` reporter — `niagara.validate level:"strict"` returned `valid:true` with an empty `errors` array for `/Game/FPS/VFX/NS_Explosion` and `/Game/FPS/VFX/NS_Smoke_Grenade`, while the same response's nested `compile.valid` was `false` and ten particle scripts across five emitters carried `compileStatus:"NCS_Error"`; `hasOutstandingCompilationRequests` and `hasActiveCompilations` were both false, so no compile was in flight. Both systems then refused to run: `effect.activate_niagara` returned `active:false` with the "NOTHING is running" warning, while the twelve sibling systems validated and activated identically-but-truthfully in the same world in the same minute. The wiki page directs callers to treat the top-level arrays as authoritative and never mentions `NCS_Error`, so a caller following the documentation cannot see the failure; the VFX stream shipped both systems as structurally verified on the strength of this verb.
- `#2-reproduces-across-an-editor-restart` `OPEN` reporter — Re-ran both validates at 2026-09-03T00:23Z in a **freshly restarted editor** (the shared instance had died on an unrelated TextureHandler crash at 22:32:57Z and come back with a clean package set, so nothing from the original session was resident). Byte-identical verdict: `NS_Smoke_Grenade` and `NS_Explosion` both return top-level `valid:true` with `errors:[]`, both carry `compile.valid:false`, and the same ten particle scripts across the same five emitters (`Smoke`/`Core`/`Wisps` and `Shockwave`/`DustRing`, Spawn+Update each) report `compileStatus:"NCS_Error"`; `hasOutstandingCompilationRequests` and `hasActiveCompilations` false, `dataInterfaceCheck:"consistent"` on both. This rules out the two readings that would have made the report benign — a stale in-memory compile result and a compile still settling after load. The error state is what is serialised on disk, and the false green is reproducible from a cold start.
- `#3-root-cause-found-the-write-that-produced-the-NCS_Error-and-the-exact-hlsl` `OPEN` builder — **The builder side of `#1`/`#2`: here is what actually broke the ten scripts, and it makes this ticket worse rather than better.** The two dead systems were mine. The cause is a PinWright write that looks entirely supported and emits invalid HLSL: **setting inputs on a dynamic input that is nested inside another dynamic input.** I had `ScaleSpriteSize`'s `Uniform Scale Factor` driven by `Lerp_Float`, with that node's `Alpha` driven in turn by `RampInOut`, and then set `RampInOut`'s own `RampIn`/`RampOut`. Every one of those `set_module_input` calls returned success.
  The generated shader, from `Saved/Logs/EAContentExamples58.log` at `2026.09.03-00.54.32`, translation unit `/Engine/Generated/NiagaraEmitterInstance.ush`:
```
/*0795*/ Context.Map.UniformScaleFactor.Lerp_Float.Alpha = RampInOut_Emitter_FuncOutput_Output;
/*0796*/ Context.Map.UniformScaleFactor.Lerp_Float.Alpha.RampInOut.RampIn  = Constant55;
/*0797*/ Context.Map.UniformScaleFactor.Lerp_Float.Alpha.RampInOut.RampOut = Constant56;

(796): error: Invalid swizzle / mask 'RampInOut'
(796): error: cannot assign value of type 'vec2' to type '': no implicit conversion allowed
(797): error: Invalid swizzle / mask 'RampInOut'
LogNiagaraCompiler: Error: 1 errors encountered compiling Vector VM shaders
```
  Line 795 assigns `Alpha` as a scalar; 796-797 then dereference that same scalar as a struct. The emitter is writing member access on a `float`, so the translation unit cannot compile and every script in the emitter lands in `NCS_Error`.
  **Why this belongs on this ticket and not only on `F-niagara-dynamic-input-nested-inputs`.** Three separate signals said the asset was fine: every `set_module_input` returned success with a value echo; `niagara.compile {force:true, wait:true}` returned `status:"completed", compiled:true` with no error field; and `niagara.validate level:"strict"` returned `valid:true, errors:[]`. The compile verb reporting `completed` for a compile that produced only errors is a fourth false-green beyond the three `#1` names, and it is the one a caller is most likely to trust because it is the verb whose entire job is compiling. Nothing short of grepping the editor log for `errors encountered compiling Vector VM shaders` surfaces it.
  **Confirmed fix and confirmed diagnosis in one step:** removing the `ScaleSpriteSize` module from the five affected emitters (`E_Explosion_Shockwave`, `E_Explosion_DustRing`, `E_Smoke_Smoke`, `E_Smoke_Core`, `E_Smoke_Wisps`), recompiling and re-adding each to its system took `compile.valid` from `false` to `true` and all six `NS_Smoke_Grenade` particle scripts from `NCS_Error` to `NCS_UpToDate`, with no new VM compile errors in the log. That confirms the nested chain was the sole cause.
  **Two asks on top of `#1`'s.** (a) `niagara.compile` should report the compile's own error list rather than `status:"completed"` when the VM shader compile produced errors — it already has them, they are in the log it wrote. (b) The nested-dynamic-input write should be refused at the point of the `set_module_input` call, since the resulting HLSL is unconditionally invalid; refusing costs nothing and the current behaviour ships a dead asset that four verbs call healthy.
  Evidence: log lines quoted above; before/after `compile.scripts[].compileStatus` on both systems; `NS_Explosion` 1452799 b and `NS_Smoke_Grenade` 772447 b re-saved after the fix.
- `#4-verified-in-fps-build` `IN-REVIEW` VFX — **`niagara.validate` now fails on `NCS_Error`, at
  both levels, with the compiler's own text. Direct route retried once per PLAN rule 2; leaving
  `IN-REVIEW` for the tester.** Live editor port 27145, UE 5.8, EAContentExamples58, 2026-09-05,
  on a scratch system built for this and deleted after.

  **Reproducing an `NCS_Error` no longer goes through `#3`'s nested-dynamic-input route** — that
  write is now refused outright (see `B-niagara-module-input-dotted-subinput-silent-noop` `#5`), so
  a different break was used: link an emitter-scope module input to a particle attribute, which
  the compiler cannot read from that namespace.
  `niagara.set_module_input {assetPath:".../NS_TR_Broken", emitter:"Bad",
  entryId:"Bad:68A8CD574D62C54866BE778FB68D9342", inputName:"Spawn Count",
  value:{link:"Particles.NormalizedAge"}}` -> `success:true, linked:true`, then
  `niagara.compile {force:true, wait:true}` -> `status:"completed"`.

  `niagara.validate {level:"basic"}` -> **`valid: false`**, `scriptCompileCheck: "failed"`, and one
  top-level `errors[]` entry:
  `NIAGARA_SCRIPT_COMPILE_ERROR` — *"Script 'NS_TR_Broken.SystemUpdateScript' is at NCS_Error: its
  last compile failed, so the engine refuses to instance this system and nothing renders. Compiler
  said: Variable Particles.NormalizedAge is in a namespace that isn't valid for reading - Node: Map
  Get - | Error compiling input for set node. - Node: Map Set Pin: SpawnBurst_Instantaneous.Spawn
  Count - ."* — carrying `scriptUsage`, `scriptPath`, `compileStatus:"NCS_Error"` and
  `compileErrors[]`. `niagara.validate {level:"strict"}` returned a **byte-identical** verdict, so
  the code is genuinely not level-escalated, which is what `#1`'s *"`basic` should raise it too"*
  asked for. The nested `compile.valid` is `false` and the top-level `valid` now agrees with it —
  the disagreement that defined this ticket is gone.

  **Regression floor.** The healthy sibling system in the same folder, compiled the same way,
  returned `valid:true`, `errors:[]` and `dataInterfaceCheck:"consistent"`; `scriptCompileCheck`
  read `"unverified"` there, never `"failed"`, so the promotion does not fire on a healthy asset.

  **`#3`(a) is still open and should not be closed with this.** `niagara.compile {force:true,
  wait:true}` on the broken system returned `{"status":"completed","compiled":true,
  "completed":true,"timedOut":false}` with no error field, for a compile that produced two errors
  and left a script at `NCS_Error`. The compile verb is still optimistic; only validate is honest
  now. **The new `niagara.compile_status` is honest and is the cheap probe `#3`(a) wanted**:
  `niagara.compile_status {assetPath:".../NS_TR_Broken"}` -> `{"status":"failed",
  "completed":true,"successful":false,"scriptCompileCheck":"failed","failedScriptCount":1}` in a
  ~350-byte response, no file spill. A caller should branch on `compile_status`, not on
  `compile`'s `status`.

  **Not verified: nothing visual, and no activation check.** `#1`'s correlation (`NCS_Error` =>
  `effect.activate_niagara` returns `active:false` => zero pixels) was not re-measured — capture
  and level work are owned by the VFX lead on this stream. `componentActivation` read
  `no_components` throughout because nothing places these scratch systems.

## Fix

Confirmed true against source before changing anything. `AddCompileIssues`
(`NiagaraInspectHandler.cpp`) is the ONLY bridge from the compile block into the top-level
verdict, and it reads `compile.issues` alone. Nothing ever wrote a script's compile STATUS into
that array: `BuildCompileDiagnosticsJson` (`NiagaraDumpBuilder.cpp`) filled `issues` with
`COMPILE_DEFERRED_ON_LOAD`, the authored structural issues, and `COMPILE_STATE_UNINITIALIZED`,
and published `compile.valid` / `compile.scripts[].compileStatus` as fields only. `FinishValidationResult`
sets `valid = Errors.Num() == 0`, so ten `NCS_Error` scripts produced `valid:true, errors:[]`
exactly as reported. Second gap found while reading: the compile-scripts array enumerated only
system spawn/update + emitter/particle spawn/update, so an event-handler, simulation-stage or GPU
compute script at `NCS_Error` was invisible even in the nested block — while
`FVersionedNiagaraEmitterData::IsValidInternal` consults exactly those.

Design: promote the statuses the response already carries, rather than re-deriving them. The
verdict is computed from the same JSON block the caller is handed, so the nested detail and the
top-level answer cannot disagree again — which is the shape of this defect. `compile.valid`
is deliberately NOT promoted: `UNiagaraSystem::IsValid()` is equally false for a system with zero
emitter handles, so promoting it would turn the documented `basic`-level `NO_EMITTERS` warning into
an unconditional error and fail validate on every freshly-created system. Both states it conflates
are reported separately. Pending compiles are reported (`pendingCompile:true` + error), not waited
out: `niagara.compile {wait:true}` already owns the bounded 90 s wait and validate is a read verb.

Response additions: `scriptCompileCheck` (`passed` / `failed` / `unverified` — published on every
verdict so "nothing compiled this yet" can never read as a pass), `pendingCompile` (systems only),
`compile.scripts[].compileErrors` on a failed script, and per-failure
`NIAGARA_SCRIPT_COMPILE_ERROR` errors at every level carrying `emitter` / `scriptUsage` /
`scriptPath` / `compileStatus` / `compileErrors`. `NCS_Dirty` and the two `…WithWarnings` statuses
stay non-fatal (the engine still instances those).

Files changed:
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraCompileVerdict.h` (new) —
  `FScriptCompileFailure` / `EScriptCompileCheck` / `FCompileVerdict` + `ReadCompileVerdict`.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraCompileVerdict.cpp` (new) —
  the pure reader over a compile block, its wire spellings, and the caller-facing failure text.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraDumpBuilder.cpp` —
  `BuildScriptJson` emits `compileErrors` (from `FNiagaraVMExecutableData::LastCompileEvents`,
  falling back to `ErrorMsg`) on `NCS_Error`; `MakeAuthoredScriptArray` strips it alongside
  `compileStatus`; new `AddEmitterCompileEntries` adds event-handler, simulation-stage and
  GPU-compute scripts to both the system and standalone-emitter compile arrays.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraInspectHandler.cpp` —
  `AddCompileStatusIssues`, called on all three validate branches (system / emitter / script).
- `Plugins/PinWright/Docs/wiki-src/niagara.md` — new `#### Did its scripts compile?` subsection
  under `### niagara.validate`: the three-state field, the two codes, `pendingCompile`, why only
  `NCS_Error` fails, and why `compile.valid` is not the field to branch on.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Niagara/TestNiagaraValidateScriptCompileError.cpp`
  (new) — `PinWright.niagara.validate.CompileVerdictReader` (pure reader: failed / passed /
  unverified / pending) and `PinWright.niagara.validate.ScriptCompileErrorFailsVerdict`
  (end-to-end on a duplicated fixture system whose particle update script is forced to `NCS_Error`).
- `Plugins/PinWright/Source/PinWright/Private/Tests/Infra/TestNiagaraValidateStrictLevelDocs.cpp` —
  added `PinWright.infra.wiki_handler.MethodPage.NiagaraValidateScriptCompile`.

Not compiled and not run here by instruction; a separate compile pass follows.

Reviewer verification:
1. Compile the plugin, then run `PinWright.niagara.validate` + `PinWright.infra.wiki_handler` —
   4 new registrations, 0 failures. `ScriptCompileErrorFailsVerdict` asserts the counterfactual
   (same system, status untouched, raises no `NIAGARA_SCRIPT_COMPILE_ERROR`), so a reverted
   promotion fails it rather than passing vacuously.
2. In a live editor, reproduce the original measurement: author a nested dynamic-input chain (see
   history `#3`) or otherwise break a module, `niagara.compile`, then
   `call("niagara.validate", {assetPath: ..., level: "basic"})`. Expect `valid:false`,
   `scriptCompileCheck:"failed"`, and one `NIAGARA_SCRIPT_COMPILE_ERROR` per failing script naming
   the emitter, the script slot and the HLSL error text. `level:"strict"` must give the same
   verdict — this code is not level-escalated.
3. Regression floor: validate a healthy stock system (e.g. a duplicate of
   `/Niagara/DefaultAssets/Templates/Systems/SimpleExplosion`) and confirm `valid` is unchanged
   from before the fix and `scriptCompileCheck` is `passed` or `unverified`, never `failed`.
4. Confirm `asset.dump` output is unchanged: the authored `compile.json` path
   (`BuildCompileJson` → `MakeAuthoredScriptArray`) still strips `compileStatus` and now also
   `compileErrors`. Note it DOES gain script entries for emitters that own event-handler /
   simulation-stage / GPU scripts — that is intended and no aspect-version bump is needed for the
   live diagnostics path, but re-dump such a system and eyeball the sidecar.

Open items for the reviewer to decide, not fixed here (each is its own ticket's scope):
- History `#3`(a): `niagara.compile` still returns `status:"completed"` for a compile that produced
  only errors. Its response is untouched by this fix; a caller now learns the truth from
  `niagara.validate`, but the compile verb itself is still optimistic.
- History `#3`(b): `set_module_input` still accepts the nested-dynamic-input write that generates
  invalid HLSL — that belongs to `F-niagara-dynamic-input-nested-inputs`.
