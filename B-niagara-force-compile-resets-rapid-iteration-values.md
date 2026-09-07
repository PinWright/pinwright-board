---
id: B-niagara-force-compile-resets-rapid-iteration-values
title: "niagara.compile {force:true} rebuilds systemUpdateRapidIteration from module template defaults, silently discarding every authored value niagara.set_parameter wrote into it"
status: OPEN
severity: High
category: bug
tags: [niagara, compile, force, rapid-iteration, set-parameter, silent-revert, data-loss, vfx]
encounters: 2
lastSeen: 2026-09-07T07:45:00Z
related: [B-module-input-override-inert-runtime-uses-rapid-iteration-default, B-niagara-set-parameter-emitter-scope-arms-di-mismatch]
---

# A forced compile reverts the rapid-iteration store to template defaults

`niagara.set_parameter {scope: "systemUpdateRapidIteration", ...}` writes and the value reads back
correctly. A subsequent `niagara.compile {force: true}` puts the module template default back, at
the same store offset, with no mention of it in the compile response.

Rapid-iteration parameters exist precisely so a value can change **without** a recompile. Resetting
them on compile inverts that contract, and it silently reverts authored content.

## Measured, live editor port 27145, UE 5.8, EAContentExamples58, 2026-09-07

`/Game/FPS/VFX/NS_Blood`:

```
niagara.set_parameter {scope:"systemUpdateRapidIteration",
    name:"Constants.Drips.SpawnBurst_Instantaneous.Spawn Count", type:"int32", value:90}
# -> success
niagara.inspect {parametersOnly:true, parameterName:"Drips.SpawnBurst_Instantaneous.Spawn Count"}
# -> value: 90            (offset 32)

niagara.compile {force:true, wait:true}
# -> status "completed", compiled true, waitedMs 3613
niagara.inspect {parametersOnly:true, parameterName:"Spawn Count"}
# -> value: 1             (offset 32)   <-- reverted, no notice
```

The regeneration is not merely a write-through of a stale copy: deleting the parameter outright and
then compiling brings it **back**.

```
niagara.remove_parameter {scope:"systemUpdateRapidIteration",
    name:"Constants.Drips.SpawnBurst_Instantaneous.Spawn Count"}      # -> success
niagara.inspect {parametersOnly:true, parameterName:"SpawnBurst_Instantaneous.Spawn Count"}
# -> Drips entry absent; Burst/Mist/Spray offsets shift (100/140/24), so the store really rebuilt
niagara.compile {force:true, wait:true}                                # -> status "completed"
niagara.inspect {parametersOnly:true, parameterName:"SpawnBurst_Instantaneous.Spawn Count"}
# -> Constants.Drips.SpawnBurst_Instantaneous.Spawn Count = 1, offset 32 again
```

## Impact

High. Combined with `B-module-input-override-inert-runtime-uses-rapid-iteration-default` — where the
rapid-iteration constant, not the override pin, is what the simulation reads — this makes the only
working lever for those inputs revert on the next forced compile by **any** agent in a shared
editor. An asset that was correct when saved silently regresses to template defaults the next time
someone recompiles it, and neither the compile response nor `niagara.validate` mentions it.

`force: false` on an up-to-date system issues no compile (`status: "notRequested"`) and is therefore
harmless; only `force: true` destroys the values.

## Workaround in use

Order of operations: all graph edits, then the **final** `niagara.compile {force:true}`, then every
`niagara.set_parameter` on a rapid-iteration scope, then `asset.save`. Never compile again before
saving, and warn downstream agents not to force-compile the asset.

## Fix direction

Preserve existing rapid-iteration values across a compile — merge on parameter name rather than
rebuilding from module template defaults — and, where a rebuild is genuinely required, report the
parameters whose values were replaced in the compile response.

## Second encounter, 2026-09-07 07:45 UTC: `force` is not the variable, and the reset is scope-selective

Two corrections to the statements above, both measured on `/Game/FPS/VFX/NS_Impact_Concrete` in the
same session.

**1. `force: false` is not safe. What is safe is a compile that does not run.** The sentence
"`force: false` on an up-to-date system issues no compile and is therefore harmless" is true only
for the up-to-date case, and a Niagara system whose emitter scripts sit at `compileStatus: null`
(the ordinary post-load state under `fx.Niagara.OnDemandCompile`) is **not** up to date. On all
eleven systems tried this session, `niagara.compile {force:false, wait:true}` returned
`requested: true, compiled: true, status: "completed"` in 40-650 ms — a real compile — and reset the
rapid-iteration values exactly as `force: true` does:

```
niagara.set_parameter {scope:"systemUpdateRapidIteration",
    name:"Constants.Chunks.SpawnBurst_Instantaneous.Spawn Count", type:"int32", value:9}   # + Sparks 2, Dust 10
niagara.compile {assetPath:"/Game/FPS/VFX/NS_Impact_Concrete", force:false, wait:true}
# -> requested:true, compiled:true, completed:true, waitedMs:89, status:"completed"
niagara.inspect {parametersOnly:true, parameterName:"SpawnBurst_Instantaneous.Spawn Count"}
# -> Chunks 1, Dust 1, Flash 1, Light 1, Sparks 1     <- all reverted
#    offsets also moved (176 -> 72, ...), so the store was rebuilt, not overwritten in place
```

So the guidance to give downstream agents is not "do not `force: true`" but **"do not compile at all
after the RI writes"**, and a caller who thinks `force: false` is a safe probe is wrong whenever the
asset has not compiled this session.

**2. Only the system-wide stores are rebuilt; the emitter-scoped ones survive.** The same compile
that reset all five `systemUpdateRapidIteration` Spawn Counts left every
`spawnRapidIteration` / `updateRapidIteration` value written moments earlier untouched:

```
# written before the compile, read back after it:
Constants.Flash.InitializeParticle.Color   = {r:7,   g:5,   b:3,   a:1}    (intact)
Constants.Light.InitializeParticle.Color   = {r:280, g:209, b:140, a:1}    (intact)
Constants.Sparks.InitializeParticle.Color  = {r:2.6, g:1.1,  b:0.3, a:1}   (intact)
Constants.Dust.InitializeParticle.Color    = {r:0.95,g:0.94, b:0.92,a:1}   (intact)
Constants.Chunks.InitializeParticle.Color  = {r:1,   g:1,    b:1,   a:1}   (never written, still template default)
```

That asymmetry makes a compile survivable, and is what the repair of the whole `/Game/FPS/VFX/`
package was built on:

1. every **emitter-scoped** write (`spawnRapidIteration` / `updateRapidIteration`),
2. **one** `niagara.compile {force:false, wait:true}` — needed anyway to clear the
   `dataInterfaceCheck: "mismatched"` those writes cause, see
   `B-niagara-set-parameter-emitter-scope-arms-di-mismatch`,
3. every **system-scoped** write (`systemUpdateRapidIteration`: Spawn Count, Loop Duration, Spawn
   Time),
4. `asset.save`, and never a compile after step 3.

Verified by re-reading the RI stores and then by searching the saved `.uasset` bytes for the packed
byte image of the whole `systemUpdateRapidIteration` store: present once on all 12 systems, while
the same image with the counts reverted to 1 and the durations to 2 is absent (0 occurrences).

## History

- `#1-force-false-also-resets` `OPEN` reporter — Second encounter, two corrections. (a) `force: false` is not harmless: on a system whose emitter scripts are at `compileStatus: null` it issues a real compile (`requested:true, compiled:true, status:"completed"`, 40-650 ms across 11 systems) and rebuilds `systemUpdateRapidIteration` from template defaults just as `force: true` does — Spawn Counts 9/2/10 came back as 1/1/1 with the store offsets shifted, proving a rebuild rather than an in-place write. The rule to publish is "no compile after the RI writes", not "no forced compile". (b) The rebuild is scope-selective: the emitter-scoped `spawnRapidIteration` / `updateRapidIteration` values written immediately before the same compile survived it intact, which makes the order emitter-scope writes -> one unforced compile -> system-scope writes -> save a working sequence. That sequence repaired 106 shadowed inputs across 12 systems under `/Game/FPS/VFX/`, verified against the `.uasset` bytes on disk (repaired store image present once; pre-fix image absent).
