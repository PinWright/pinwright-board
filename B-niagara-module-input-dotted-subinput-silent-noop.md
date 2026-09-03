---
id: B-niagara-module-input-dotted-subinput-silent-noop
title: "niagara.set_module_input accepts any dotted sub-input name and writes an unread rapid-iteration parameter — affirmative success for a no-op"
status: OPEN
severity: High
category: bug
tags: [niagara, set-module-input, dynamic-input, rapid-iteration, silent-noop, success-no-effect]
encounters: 3
lastSeen: 2026-09-02T22:59:00+05:00
---

# `niagara.set_module_input` validates nothing after the first dot: an invented sub-input name succeeds and writes a parameter nothing reads

`niagara.set_module_input` resolves `inputName` against the addressed module and,
when no override pin exists, falls back to writing a **rapid-iteration parameter**
named `Constants.<Emitter>.<ModuleFunctionName>.<inputName>` (the response reports
`kind: "rapid"` on the matching `reset_module_input`). That name is built by string
concatenation with **no check that `inputName` is one of the module's declared
inputs**. Any string is accepted, including a dotted path that names nothing.

The caller gets the full success envelope — `success:true`, a fresh `pinId`, an
`index`, and the canonical `value` echo the wiki explicitly tells callers to trust
("Confirm a literal edit from this result directly — no second inspect needed") —
for a write that the compiled script can never read.

## Repro (live, UE 5.8, PinWright port 27145, 2026-09-02)

Emitter `E_ImpConcrete_Sparks` = `asset.duplicate` of
`/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst`, with a
`RandomRangeInt` dynamic input assigned to `SpawnBurst_Instantaneous.Spawn Count`
(entryId `68A8CD57…`). The dynamic input's own inputs are `Minimum` / `Maximum`
(confirmed via `niagara.inspect {includeStack:true}` on the `RandomRangeInt`
stack entry).

Two *different*, *mutually exclusive*, *both invented* spellings were sent one
after the other. Both returned success:

```
niagara.set_module_input {entryId:"68A8CD57…", inputName:"Spawn Count.Minimum", value:8}
  -> {"success":true, …, "inputName":"Spawn Count.Minimum",
      "pinId":"5FB2A608…","index":1,"linked":false,"value":"8.0"}

niagara.set_module_input {entryId:"68A8CD57…", inputName:"Spawn Count.RandomRangeInt.Minimum", value:8}
  -> {"success":true, …, "inputName":"Spawn Count.RandomRangeInt.Minimum",
      "pinId":"2BE12966…","index":1,"linked":false,"value":"8.0"}
```

They cannot both be right, and in fact neither is: Niagara aliases a dynamic
input's own inputs by the **dynamic-input node's** function name, not by the
parent module + input path — the template's own pre-existing dynamic input proves
the shape, appearing in `spawnRapidIteration` as
`Constants.<Emitter>.FloatFromCurve001.Scale Curve` (no owning-module segment).
The correct name here would be `Constants.<Emitter>.RandomRangeInt.Minimum`.

`niagara.reset_module_input` then removed both, reporting `reset:true`,
`previousValue:"8.0"`, `kind:"rapid"` — confirming the writes had landed in the
rapid-iteration store under the two bogus names.

## Why it matters

- **It is a silent false success on the namespace's most-used write verb.** The
  wiki's stated verification protocol for a literal write is "trust the echo".
  The echo is exactly what this defect fabricates, so the documented check cannot
  catch it.
- **It is the first thing a caller reaches for.** Setting a dynamic input's own
  inputs is unimplemented (`F-niagara-dynamic-input-nested-inputs`), and
  addressing the dynamic-input node directly answers `INVALID_STACK`. A dotted
  sub-path is the natural next guess — and it answers `success:true`, so the
  caller stops looking and ships an effect whose random range never took.
- **It pollutes the asset.** Each bogus name is a real entry written into the
  emitter's rapid-iteration store and serialized on save; nothing removes it
  unless the caller happens to replay the same invented string through
  `reset_module_input`.

## Fix

Validate `inputName` against the module's declared stack-function inputs before
taking the rapid-iteration path, and reject an unknown name with a distinct code
(e.g. `MODULE_INPUT_NOT_FOUND`) that lists the module's real input names. The
enumeration needed is the same one `F-niagara-dynamic-input-nested-inputs`
identifies (`FNiagaraStackGraphUtilities::GetStackFunctionInputs` with an
`FCompileConstantResolver`); until that lands, a cheaper guard is to refuse any
`inputName` containing a `.` on the rapid path, since a module's own input names
never contain one — that alone closes the observed case without new machinery.

severity rationale: impact=silent-false-success on the namespace's primary write verb, with the documented verification method unable to detect it × reach=every caller who tries to configure a dynamic input (the standard way to randomize any module input) -> High

## History
- `#1-initial-repro` `OPEN` reporter — Hit while authoring `/Game/FPS/VFX/NS_Impact_Concrete` in the EAContentExamples58 checkout on UE 5.8. Assigned `RandomRangeInt` to `SpawnBurst_Instantaneous.Spawn Count` via `{dynamicInput:…}`, then tried to set its `Minimum`. Addressing the dynamic-input node's own `entryId` gave `INVALID_STACK` (that is `F-niagara-dynamic-input-nested-inputs`); the dotted fallbacks `"Spawn Count.Minimum"` and `"Spawn Count.RandomRangeInt.Minimum"` **both** returned `success:true` with a `pinId` and a `"8.0"` value echo, for two mutually exclusive invented names — proving no validation. `reset_module_input` reported `kind:"rapid"`, `previousValue:"8.0"` for both, confirming real writes into the rapid-iteration store under names nothing reads (the correct alias would be `Constants.<Emitter>.RandomRangeInt.Minimum`, per the template's own `Constants.<Emitter>.FloatFromCurve001.Scale Curve`). Both bogus entries were removed by the reporter. Cheap guard available today: refuse a rapid-path `inputName` containing a `.`.
- `#2-graph-level-counterexample-narrows-the-claim` `OPEN` reporter — **Different reporter, same day, contradicting measurement on a different module — this does not refute `#1`, it narrows it.** 2026-09-02, UE 5.8, EAContentExamples58 checkout, live editor on port 27145, while authoring `/Game/FPS/VFX/NS_Impact_Dirt`. On emitter `E_ImpactDirt_Spray` (`SimpleSpriteBurst` duplicate) I assigned `{dynamicInput:"/Niagara/DynamicInputs/Lerp/Lerp_Float"}` to `ScaleSpriteSize.Uniform Scale Factor` (a **NiagaraFloat** input, module added by `niagara.add_module`, entryId `6BAEC061…`), then wrote the dotted sub-inputs `"Uniform Scale Factor.A"` = 1, `"Uniform Scale Factor.B"` = 2.1, assigned a further nested dynamic input at `"Uniform Scale Factor.Alpha"` = `RampInOut`, and wrote `"Uniform Scale Factor.Alpha.Ramp In"` = 1. All four returned the same success envelope `#1` describes. **But these landed as real graph override pins, not rapid-iteration parameters.** `niagara.inspect {includeGraphs:true}` on the emitter shows, in the ParticleUpdate stack graph: `ScaleSpriteSize.Uniform Scale Factor` (`linkCount:1`, driven by the `Lerp Float` function-call node), `Uniform Scale Factor.A` `defaultValue:"1.0"` `linkCount:0`, `Uniform Scale Factor.B` `defaultValue:"2.1"`, `Uniform Scale Factor.Alpha` `linkCount:1` (driven by the `Ramp in Out` node), and `Uniform Scale Factor.Alpha.Ramp In` `defaultValue:"1.0"` — i.e. the `<Input>.<SubInput>` naming the Niagara editor itself uses on the override `ParameterMapSet`, two levels deep. Nothing named `Constants.…` was involved. So on this module the dotted spelling is not a no-op and not a bogus rapid write. What differs from `#1`: float input vs **int** (`Spawn Count`), `ScaleSpriteSize` in ParticleUpdate vs `SpawnBurst_Instantaneous` in EmitterUpdate, `Lerp_Float`/`RampInOut` vs `RandomRangeInt`. Any of those could gate which path the handler takes. **Not done here:** I did not run `reset_module_input` to read back `kind`, and I did not re-run `#1`'s exact `Spawn Count.Minimum` call, so I cannot say which factor decides — and `#1`'s `kind:"rapid"` readback is stronger evidence for its own case than my graph read is against it. Second sub-input pair from the same session, `"Velocity Strength.Minimum"`/`".Maximum"` on `AddVelocityInCone` (`RandomRangeFloat` assigned), was **not** graph-verified — echo only. Suggested next step for whoever fixes this: before adding `#1`'s "refuse any `inputName` containing a dot" guard, check that it does not also refuse the working two- and three-segment paths above, which are currently the only route to a dynamic input's own inputs and the only route to over-life size scaling (see `B-niagara-set-curve-keys-unreachable-module-input-di` and `B-niagara-module-input-link-particle-attribute`, both filed this session). A guard that keys off "the resolver found a real override pin" rather than off the dot would separate the two cases.
- `#3-third-combination-bogus-names-reach-disk-and-breakExistingLink-leaves-the-chain` `OPEN` reporter — Third reporter, same day, UE 5.8, EAContentExamples58, live editor port 27145, building the FPS blood / smoke-grenade / explosion Niagara systems under `/Game/FPS/VFX/`. Four additions, one of which corroborates `#2` against `#1`. **(a) A third module/dynamic-input combination behaves like `#1`'s echo and like `#2`'s pins: unvalidated.** On `AddVelocityInCone` (ParticleSpawnScript, added by `add_module`) with `{dynamicInput:"/Niagara/DynamicInputs/UniformRange/UniformRangedFloat.UniformRangedFloat"}` assigned to `Velocity Strength`, `"Velocity Strength.Minimum"`=500 and `".Maximum"`=1400 both returned the full success envelope. I then wrote a deliberately invented `"Uniform Scale Factor.NoSuchInput"`=5 on a `Lerp_Float`-driven `ScaleSpriteSize` — **also `success:true`, with a fresh `pinId` and a `"5.0"` echo**, confirming `#1`'s core claim on a module where `#2` shows the dotted path otherwise works. There is no safety net on either path. **(b) The bogus names reach disk and survive.** After `niagara.compile {force:true}` + `asset.save {force:true}`, `grep -a "Velocity Strength\.[A-Za-z]*"` on `Content/FPS/VFX/Emitters/E_Blood_Spray.uasset` returns both `Velocity Strength.Minimum` and `Velocity Strength.Maximum`, so this is not an in-memory artefact — a wrong spelling ships. **(c) A rapid-store readback cannot arbitrate between the spellings, which corroborates `#2` and further narrows `#1`.** I wrote three candidate spellings on one `Lerp_Float`-driven `ScaleSpriteSize.Uniform Scale Factor` — `"Uniform Scale Factor.B"`, `"ScaleSpriteSize.Uniform Scale Factor.Lerp_Float.Alpha"`, and the invented one — then compiled and read BOTH rapid-iteration scopes in full (`niagara.inspect {parametersOnly:true, parameterName:"Constants"}`, 13 spawn + 9 update entries listed): **none of the three produced any entry**, and `parameterName:"Lerp"` returns nothing in either scope. That matches `#2`'s finding that this module's sub-inputs land as graph override pins rather than rapid parameters, and it means `#1`'s `reset_module_input {kind:"rapid"}` readback — the strongest arbitration signal on the board — is unavailable on ParticleUpdate float inputs. A caller has no published way to tell a live sub-input from a dead one without simulating the system. **(d) `breakExistingLink:true` does not remove the chain it says it replaced.** `niagara.set_module_input {inputName:"Velocity Strength", value:1350, breakExistingLink:true}` returned `replacedOverride {valueMode:"dynamicInput", source:"/Niagara/DynamicInputs/UniformRange/UniformRangedFloat.UniformRangedFloat"}` and the literal does govern — but after compile + save, the saved `.uasset` still contains three `UniformRangedFloat` references and both orphaned `Velocity Strength.Minimum` / `.Maximum` pin names. The wiki's "deletes the link and its now-orphaned upstream chain first" is only half true; the pins survive. **Consequence for this package:** with `#1`, `#2` and this entry disagreeing on which spelling is live, and no non-simulating way to check, I abandoned dynamic inputs entirely across 19 emitters and fell back to the `InitializeParticle` `Random` static-switch modes plus `AddVelocityInCone`'s `Use Velocity Falloff On Cone Axis` for per-particle speed spread. Per-particle random ranges remain effectively unavailable, as `F-niagara-dynamic-input-nested-inputs` `#2` also argues.
- `#3-retracting-my-own-counterexample-2-was-wrong` `OPEN` reporter — **Retracting `#2` in full. Same reporter, same session, one hour later: `#1` is right and my counterexample was a misread.** The decisive test `#2` said it had not done, I then did: `niagara.reset_module_input {assetPath:"/Game/FPS/VFX/Emitters/E_ImpactDirt_Dust", entryId:"4A3FBE55…"(AddVelocityInCone), inputName:"Velocity Strength.Minimum"}` returned `reset:true, previousValue:"60.0", wasLinked:false, **kind:"rapid"**` — a **float** input on a ParticleSpawn module with `RandomRangeFloat` assigned, i.e. exactly the case `#2` claimed was different from `#1`'s int-on-EmitterUpdate one. It is not different. The dotted sub-input writes land in the rapid-iteration store under a name nothing reads, on every module I tried. What misled me: `niagara.inspect {includeGraphs:true}` renders those rapid-iteration entries as input nodes carrying their **short** name (`Velocity Strength.Minimum` = `"600.0"`, `Uniform Scale Factor.A` = `"1.0"`), sitting in the same node list as the genuine module override pin (`ScaleSpriteSize.Uniform Scale Factor`, `linkCount:1`). I read the short-named nodes as override pins because they looked identical in that view and because they survived a save, an editor crash and a reload — persistence proves serialisation, not that anything reads them. **So my `#2` advice to spare dotted paths from `#1`'s guard should be ignored: implement the guard.** The `<Input>.<SubInput>` spelling has no legitimate use I found. Worth adding to the fix: the graph aspect should distinguish a rapid-iteration input node from an override pin, since as rendered today it is what makes this defect look like a working feature to a caller who follows the wiki's own advice to read the effective value from the graphs aspect. Practical cost on my side: every `RandomRangeFloat` and every `Lerp_Float`/`RampInOut` chain across thirteen FPS impact emitters was inert, and all of them were torn out and rebuilt from literals plus `Mass Mode:Random` + `Drag{Ignore Mass:false}` + `Velocity Falloff Away From Cone Axis`.
