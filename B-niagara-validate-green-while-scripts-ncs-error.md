---
id: B-niagara-validate-green-while-scripts-ncs-error
title: "niagara.validate reports valid:true with zero errors while compile.valid is false and particle scripts are NCS_Error — the system then refuses to activate"
status: OPEN
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
