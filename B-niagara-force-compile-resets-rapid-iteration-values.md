---
id: B-niagara-force-compile-resets-rapid-iteration-values
title: "niagara.compile {force:true} rebuilds systemUpdateRapidIteration from module template defaults, silently discarding every authored value niagara.set_parameter wrote into it"
status: OPEN
severity: High
category: bug
tags: [niagara, compile, force, rapid-iteration, set-parameter, silent-revert, data-loss, vfx]
encounters: 1
lastSeen: 2026-09-07T07:25:00Z
related: [B-module-input-override-inert-runtime-uses-rapid-iteration-default]
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
