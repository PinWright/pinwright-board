---
id: B-niagara-set-curve-keys-unreachable-module-input-di
title: "niagara.set_curve_keys cannot address a curve data interface that lives on a stack module input — every scope/name spelling returns DATA_INTERFACE_NOT_FOUND, so no over-life curve can be authored"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, set-curve-keys, data-interface, curve, module-input, scale-sprite-size, scale-color, discoverability, static-switch, silent-bypass, correction]
encounters: 8
lastSeen: 2026-09-03T09:40:00+05:00
---

# `niagara.set_curve_keys` addresses parameter stores only, and stack-module curve inputs are not in one

> **CORRECTION, see `#5`.** The verb defect below is real and unchanged: `set_curve_keys` cannot
> address a stack-module curve DI. But the *consequence* this ticket draws from it — that
> size-over-life "cannot be authored at all" — is **false**, and a working route exists that needs
> no curve DI. The paragraphs marked below are superseded; the recipe is in `#5`.


`niagara.set_curve_keys` takes `scope` + `parameterName` and looks the curve DI up
in that parameter store. The curves that matter in real content are **module
inputs**, not store entries:

| where the curve lives | example |
|---|---|
| `ScaleSpriteSize.Uniform Curve Sprite Scale` | size over life — the only non-curve mode is a constant factor |
| `ScaleMeshSize.Uniform Curve Mesh Scale` | mesh size over life |
| `ScaleColor.Scale Alpha` -> `FloatFromCurve.FloatCurve` | the alpha fade every stock burst template ships with |

`niagara.inspect {includeStack:true}` reports these as
`moduleInputs[] { type: "NiagaraDataInterfaceCurve", isDataInterface: true }` —
`valueMode: "default"` for the first two, `valueMode: "data"` for the
`FloatFromCurve` one, i.e. a real DI instance. Neither is reachable:

```
niagara.set_curve_keys { assetPath: "<emitter>", scope: "updateRapidIteration",
  name: "Constants.E_ImpactDirt_Spray.ScaleSpriteSize.Uniform Curve Sprite Scale", keys: [...] }
-> [DATA_INTERFACE_NOT_FOUND] Parameter '...' was not found in scope 'updateRapidIteration'.

niagara.set_curve_keys { ..., name: "Constants.E_ImpactDirt_Spray.FloatFromCurve001.FloatCurve" }
-> [DATA_INTERFACE_NOT_FOUND] ... not found in scope 'updateRapidIteration'.

niagara.set_curve_keys { ..., name: "E_ImpactDirt_Spray.FloatFromCurve001.FloatCurve" }
-> [DATA_INTERFACE_NOT_FOUND] ... not found in scope 'updateRapidIteration'.
```

The `Constants.<Emitter>.<Module>.<Input>` naming is the one
`niagara.inspect {parametersOnly:true}` publishes for that emitter's scalar
rapid-iteration entries, so it is the only naming a caller has to go on — and DIs
are not in that list. There is no verb that enumerates the DI parameters a scope
holds, so the caller cannot discover a name that would work, and cannot tell
"wrong name" from "wrong scope" from "not addressable at all".

## Why it matters

Size-over-life and colour-over-life are curve-driven in Niagara by construction:
`ScaleSpriteSize`'s `Scale Sprite Size Mode` offers `Uniform` (a constant factor)
or `Uniform Curve` (the DI). Choosing `Uniform` gives a constant — no animation —
so with the DI unreachable there is **no way to author a size ramp** [SUPERSEDED by `#5`: the
`Uniform` branch accepts a dynamic-input chain, which animates without the DI], which the
brief for these impact effects asks for explicitly (`~20 -> ~110` dust haze,
`~10 -> 140` water ring).

Combined with `B-niagara-module-input-link-particle-attribute` (cannot link
`Particles.NormalizedAge`) and `F-niagara-dynamic-input-nested-inputs` (cannot set
a nested dynamic input's static switches), all three documented routes to
over-life behaviour are closed. The one that still works is narrow and
accidental: assign `Lerp_Float` to the input, set `A`/`B` by compound path, and
assign `RampInOut` to `Alpha` — which works only because `RampInOut` reads
`Particles.NormalizedAge` internally, and only in whatever branch its `Mode`
switch already happens to sit in, since that switch cannot be set.

## Repro

1. `asset.duplicate` `/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst`
   to a scratch path. The template ships `ScaleColor` with a `FloatFromCurve001`
   dynamic input on `Scale Alpha` whose `FloatCurve` reads `valueMode: "data"`.
2. `niagara.add_module` `/Niagara/Modules/Update/Size/ScaleSpriteSize`,
   `scriptUsage: "ParticleUpdateScript"`.
3. `niagara.set_curve_keys` with any of the three `name` spellings above, against
   `scope: "updateRapidIteration"` (or `spawnRapidIteration`).
4. **FAIL** `DATA_INTERFACE_NOT_FOUND` every time.

## Expected

Either

- `set_curve_keys` accepts a **module-input target** — `entryId` + `inputName`,
  the same addressing `set_module_input` uses — creating the override DI on the
  pin if the input is still at `valueMode: "default"`; or
- a companion read verb enumerates the DI parameters each scope actually holds,
  with the exact `parameterName` strings `set_curve_keys` will accept, so the
  store route becomes usable instead of guessable.

The first is what content authoring needs; the second is the minimum that makes
the current error honest.

## Distinct from related tickets

- `F-niagara-curve-authoring` (DONE) delivered `set_curve_keys` /
  `bind_curve_asset` against parameter stores. This ticket is that the curves in
  stock modules are not in a parameter store, so the delivered verb cannot reach
  the case the feature was motivated by.
- `B-niagara-set-parameter-emitter-scope-unreachable` is the emitter-scope
  argument-schema gap on `set_parameter`; a different verb and a different cause,
  though a fix there may make the store route testable.

severity rationale: impact=hard blocker — size-over-life and curve-driven
colour-over-life cannot be authored at all, and the sibling workarounds are
blocked by two other open tickets x reach=every effect with a size or colour
ramp -> High
[SUPERSEDED by `#5`: not a hard blocker. A working route exists. The residual
defect is the unreachable DI plus a static switch that silently bypasses an
authored chain, which argues Medium on impact and High on the silent-bypass
discoverability. Left at High for the original reporter to re-judge.]

## Fix

Confirmed TRUE against source before changing anything. `niagara.set_curve_keys` built its target
with `Spec.Kind = ENiagaraEditTargetKind::ParameterStore` and looked the DI up with
`ResolveBoundDataInterface(*Target.ParameterStore, ...)`
(`NiagaraCurveHandler.cpp` — target spec, store lookup, `DATA_INTERFACE_NOT_FOUND` refusal). No
other path existed, so a curve on a module's override pin, or still at its module script's default,
was unreachable by construction. `#3`'s reading was the correct one.

**Shape of the fix.** Both curve verbs (and a new read verb) now take a second addressing form,
`entryId` + `inputName` — the same one `niagara.set_module_input` takes, `entryKey` included. The
(asset, emitter, stage, module) half is NOT duplicated: it is `NiagaraEdit::ResolveTarget` with
`ENiagaraEditTargetKind::Module`, exactly as `set_module_input` resolves it. The only new logic is
one shared "module input -> DI object" resolver.

The write rule is what makes it safe. An input at `valueMode: "default"` resolves to the module
ASSET's own object, shared by every placement of that module — engine content included — so a write
there would edit `/Niagara/Modules/...` itself. `EResolveMode::WriteCreateOverride` therefore creates
the override pin and its own DI first (via the engine's
`FNiagaraStackGraphUtilities::SetDataInterfaceValueForFunctionInput`), **seeded from the script
default** so a one-channel edit keeps the authored shape of the rest. `EResolveMode::Read` hands the
shared object back marked `writable: false`. An input driven by a dynamic input / linked parameter /
expression is refused `MODULE_INPUT_OVERRIDE_LINKED` naming the driver; an unknown `inputName` is
refused `MODULE_INPUT_NOT_FOUND` **listing the module's data-interface inputs**, which answers `#1`'s
"the caller cannot discover a name that would work".

`niagara.get_curve_keys` is new and closes the read half `#2`/`#6`/`#7`/`#8` are blocked by: it emits
the samples no other surface does, in both addressing forms, all channels by default. The store-form
`DATA_INTERFACE_NOT_FOUND` message now names the module-input form, so the refusal `#1`-`#3` cycled
against is self-correcting.

**Not addressed here, deliberately.** `#4`/`#5`'s silent static-switch bypass (an authored
dynamic-input chain on a branch the switch does not select) is a `niagara.validate` warning about a
different verb — it wants its own ticket. `#5`'s "size-over-life is authorable via `Lerp_Float`"
correction stands and is unaffected; this makes the direct route work as well.

**Files changed** (all under `Plugins/PinWright/`):
- `Source/PinWright/Private/Handlers/Niagara/NiagaraModuleInputDataInterface.h` / `.cpp` — NEW. The
  single module-input -> DI resolver, plus the reflection read of `UNiagaraNodeInput`'s private
  `DataInterface` UPROPERTY (`UCLASS(MinimalAPI)`, so `GetDataInterface()` cannot be linked).
- `Source/PinWright/Private/Handlers/Niagara/NiagaraCurveHandler.cpp` — dual addressing on
  `set_curve_keys` and `bind_curve_asset`, new `get_curve_keys`, one shared channel table behind
  both the write and the read path, self-contained payload parsing (the verbs no longer route
  through `ParseDataInterfacePayload`, whose four extra keys were undeclared and therefore dead).
- `Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp` — the duplicated reflection
  read in `RemoveOverrideValueNode` now calls the shared helper. No behaviour change.
- `Source/PinWright/Private/Handlers/ErrorCodes.h` — registered `MODULE_INPUT_NOT_FOUND` (new) and
  `CHANNEL_MISMATCH` (emitted since the original curve feature, never registered; the registry test
  missed it because it was only ever emitted from inside a ternary).
- `Source/PinWright/Private/Tests/Infra/TestDeclaredParamCoverage.cpp` — pruned the 8 now-stale
  read-but-undeclared baseline pairs for the two curve verbs (ratchet goes down, not up).
- `Docs/wiki-src/niagara.md` — `### niagara.set_curve_keys` and `### niagara.get_curve_keys` overlay
  sections: the two forms, why no store holds a module curve, the override-creation rule, the read
  gap this closes.
- `Source/PinWright/Private/Tests/Niagara/TestNiagaraCurveHandler.cpp` — 4 new tests (round-trip
  write+read through the module-input form on a real `ScaleSpriteSize` placement, the
  discoverability of `MODULE_INPUT_NOT_FOUND`, the store-miss message naming the module form, and
  the two forms being mutually exclusive).
- `Source/PinWright/Private/Tests/Infra/TestNiagaraCurveModuleInputDocs.cpp` — NEW, 2 doc-contract
  tests over the rendered method pages.

**Not compiled and not run** — a separate compile pass follows.

**Reviewer verification.** On a live editor: duplicate `SimpleSpriteBurst`, `niagara.add_module`
`/Niagara/Modules/Update/Size/ScaleSpriteSize` into `ParticleUpdateScript`, then
`niagara.set_curve_keys {assetPath, emitter, entryId: <the module's entryId>, inputName: "Uniform
Curve Sprite Scale", keys: [{time:0,value:0.2},{time:0.35,value:1.0},{time:1,value:0.05}]}`. Expect
success with `addressing: "moduleInput"`, `createdOverride: true`, `valueMode: "data"`, `written: 3`.
Read it back with `niagara.get_curve_keys` on the same address and compare the keys. Then confirm
the write did NOT touch the module asset: `niagara.get_curve_keys` against a SECOND fresh
`ScaleSpriteSize` placement must still report `valueMode: "default"` with the stock linear 0->1 ramp
`#2` read out of the binary. The sixteen emitters `#4` names are the content-level regression suite;
`E_Explosion_Shockwave` and `E_Explosion_DustRing` are the two that cannot be called correct until
this works.

## History
- `#1-initial-repro` `OPEN` reporter — Found building the FPS impact VFX systems under `/Game/FPS/VFX/` (map as forcing function; host `CLAUDE.md` § "What this project is for"), 2026-09-02, UE 5.8, EAContentExamples58 checkout, live editor on port 27145. All three failing calls above were executed against a fresh `SimpleSpriteBurst` duplicate and returned the quoted `DATA_INTERFACE_NOT_FOUND` text. The `Constants.<Emitter>.<Module>.<Input>` naming was taken from that same emitter's `niagara.inspect {parametersOnly:true}` output, which lists nine such scalar entries per scope and no data-interface entries. Not source-confirmed: no read of the `set_curve_keys` handler was made; the "parameter stores only" reading is from the verb's own parameter list (`scope` + `parameterName`) and the uniform error. Scopes tried: `updateRapidIteration` only for two spellings, plus one at the same scope with the `Constants.` prefix dropped — `spawnRapidIteration`, `rendererBindings` and `user` were **not** tried and a verifier should, though none of them is where a ParticleUpdate module input would live.
- `#2-both-rapid-scopes-measured-empty-and-the-default-curve-read-out-of-the-binary` `OPEN` reporter — Second reporter, same day, building `/Game/FPS/VFX/NS_Blood`, `NS_Impact_Flesh`, `NS_Smoke_Grenade` and `NS_Explosion` in the EAContentExamples58 checkout on UE 5.8, live editor port 27145. Two things `#1` left open, now measured. **(a) The untried scopes are not where it is hiding.** `#1` notes it only tried `updateRapidIteration` and asks a verifier to try the others. I read BOTH rapid-iteration stores in full on a probe emitter (a `SimpleSpriteBurst` duplicate carrying `add_module`-added `ScaleSpriteSize`, `ScaleColor`, `CurlNoiseForce`, `SubUVAnimation`, `Light_Attributes`, `Collision` and eleven more) with `niagara.inspect {parametersOnly:true, parameterName:"Constants"}`: `spawnRapidIteration` holds 13 entries and `updateRapidIteration` 9, and **every one is a scalar / Vector3f / Vector4f / Quat4f / bool — there is not one data-interface entry in either scope**, on an emitter that demonstrably owns at least three curve DIs. `rendererBindings` came back empty on the same read. So the DI genuinely does not live in a parameter store and no spelling can reach it; `set_curve_keys`'s `scope` + `parameterName` contract simply cannot address a module-pin DI. `niagara.set_curve_keys {scope:"updateRapidIteration", parameterName:"Constants.E_ZZ_ProbeScratch.ScaleSpriteSize.Uniform Curve Sprite Scale"}` returned the same `DATA_INTERFACE_NOT_FOUND` `#1` quotes. **(b) There is no read path either** — `niagara.decompile_model` on the same emitter (186 KB payload) mentions the DI's type, node and enum options but publishes no curve keys, so a caller cannot even see what the curve currently is, let alone edit it. **What the default curve actually is**, since the API will not say: I parsed it out of `Modules/Update/Size/ScaleSpriteSize.uasset` directly, scanning for float32 runs in [0,1] — the baked `ShaderLUT` for `Uniform Curve Sprite Scale` is a **monotone linear 0 -> 1 over `Particles.NormalizedAge`** (57 samples, 0.0, 0.018, 0.036 ... 0.982, 1.0). **Practical cost, which `#1` does not state:** because the keys cannot be edited and `Uniform Curve Scale` is only a multiplier on that 0->1 ramp, a size-over-life ramp authored through this API can only run **0 -> max**. The ordinary VFX ask — smoke growing 30 -> 260, a fireball 60 -> 300 — is unauthorable; every sprite must be born at size zero. Four systems in this package ship with that compromise. Not source-confirmed: no read of the `set_curve_keys` handler.
- `#3-third-reporter-same-wall-with-shipped-emitters-named` `OPEN` reporter — Third independent reporter, same day, building `/Game/FPS/VFX/NS_Impact_Concrete`, `NS_Impact_Metal` and `NS_Impact_Wood` in the EAContentExamples58 checkout on UE 5.8. Confirms `#1`'s mechanism verbatim from a cold start: `niagara.set_curve_keys {scope:"updateRapidIteration", parameterName:"Constants.E_ImpConcrete_Dust.ScaleSpriteSize.Uniform Curve Sprite Scale", keys:[{time:0,value:0.22},{time:0.18,value:0.75},{time:1,value:1}]}` returned `[DATA_INTERFACE_NOT_FOUND] Parameter '…' was not found in scope 'updateRapidIteration'`, and `niagara.inspect {includeStack:true}` shows that module input as `{"name":"Uniform Curve Sprite Scale","type":"NiagaraDataInterfaceVector2DCurve"/"NiagaraDataInterfaceCurve","valueMode":"default"}` — there is no DI **instance** to address, only the module script's own default object, which is why every scope/name spelling must fail rather than one of them being the right spelling. The same emitter's template-inherited `FloatFromCurve001.FloatCurve` reads `valueMode:"data"` in the very same dump, which is the contrast that makes the rule legible: a curve reachable by this verb is one something already overrode. Three shipped emitters are affected and are the reproduction targets, each left on the stock default LUT with only `Uniform Curve Scale` authored: `E_ImpConcrete_Dust` (5.0), `E_ImpWood_Dust` (4.5), `E_ImpMetal_Smoke` (3.5). Consequence recorded for the fixer: with the stock default being a linear 0->1 ramp over normalized age (read out of the module binary by the `#2` reporter), these dust puffs are born at size 0 and reach 60-85 / 45-68 / 28-49; the brief asked for 14 -> 70, so the top end is authorable through `Uniform Curve Scale` and the low end is not authorable at all. `niagara.add_data_interface` into the same rapid scope was considered as a workaround and deliberately **not** attempted: a DI resolved into a store the compiled scripts do not reference is the orphan condition that `niagara.validate`'s `dataInterfaceCheck` calls `mismatched`, which the wiki documents as arming a VectorVM assert that kills the editor. If a fix lands via that route it needs the compiled-script binding proven, not just the store write.
- `#4-measured-visually-and-the-module-has-now-been-removed-from-16-emitters` `OPEN` reporter — Fourth report, and the first with a **visual** measurement rather than an API one, plus the disposition the previous three left open. `#2` predicted the cost analytically ("every sprite must be born at size zero"); the VFX lead then captured `NS_Impact_Concrete` against a dim wall at 20 ms, 40 ms, 100 ms and 600 ms and the `Dust` emitter contributed **not one visible pixel at any timestamp**, while the `Sparks` emitter in the same system read correctly in the same frames. So the compromise is not a degraded ramp, it is an invisible emitter: a particle at size 0 that reaches full size exactly as `ScaleColor`'s alpha ramp finishes fading it out is never simultaneously large and opaque, and nothing reaches the screen. **Sixteen** emitters shipped in `/Game/FPS/VFX/` with that shape, not the four `#2` counted and not the three `#3` named: `E_ImpConcrete_Dust`, `E_ImpMetal_Smoke`, `E_ImpWood_Dust`, `E_FPS_MuzzleAR_Smoke`, `E_FPS_MuzzlePistol_Smoke`, `E_FPS_ShellSmoke`, `E_ImpactFlesh_Mist`, `E_ImpactFlesh_Puff`, `E_Blood_Mist`, `E_Smoke_Smoke`, `E_Smoke_Core`, `E_Smoke_Wisps`, `E_Explosion_Fireball`, `E_Explosion_Shockwave`, `E_Explosion_SmokeColumn`, `E_Explosion_DustRing` — across nine of the package's fourteen systems, i.e. roughly half the effects in the package rendered nothing where dust or smoke was called for.

  **What we did instead, so the next reader does not re-derive it.** There is no workaround inside `ScaleSpriteSize`, so the module was **removed outright** from every affected emitter and a real birth size authored on `InitializeParticle` (`Sprite Size Mode = Random Uniform`, `Uniform Sprite Size Min`/`Max` set to the middle-to-upper part of the range the curve had been scaling toward — e.g. concrete dust 12/17 with `Uniform Curve Scale` 5.0, i.e. an intended 0→60/85, became a flat 30/62). That is not a fix for this ticket, it is the amputation: **the package now has no size-over-life anywhere**, and two effects lose something real by it — `E_Explosion_Shockwave` and `E_Explosion_DustRing` are expanding rings by definition and now pop to a random size and hold it. Per-particle randomisation keeps successive shots from looking identical, which is the only part of the intent that survived. Worth recording because it sets the bar for the fix: once `set_curve_keys` can address a module-input DI, these sixteen emitters are the regression suite, and two of them cannot be called correct until it can.

  One correction to `#3`'s framing, offered gently: it reads the two `Uniform` / `Uniform Curve` options as a choice, but on the emitters authored by the muzzle stream the `Uniform Scale Factor` pin carries a `Lerp_Float` dynamic-input chain (`A` = 1.0, `B` = 4.6) **and** `Uniform Curve Scale` = 5.0 is set at the same time, with `Scale Sprite Size Mode` on the curve branch — so the `Lerp_Float` workaround `#2` describes was authored and then silently not read, because the static switch selects the other branch. That is the shape of the trap: the workaround leaves visible evidence in the asset that looks like an authored ramp, and nothing in `niagara.inspect` says the pin is on the unselected branch. A fix that only teaches `set_curve_keys` the module-input target still leaves that mis-read available; naming the selected branch in the stack dump would close it. Not source-confirmed: read from `asset.dump` `nir.txt` for the two muzzle emitters, not from the compiled script.
- `#5-correction-size-over-life-IS-authorable-and-the-real-defect-is-a-silent-switch-bypass` `OPEN` reporter — **Correcting this ticket's central consequence, which is false, and giving the working recipe.** The verb defect stands: `set_curve_keys` still cannot address a stack-module curve DI, and every scope/name spelling still returns `DATA_INTERFACE_NOT_FOUND`. What is wrong is the conclusion `#1`-`#4` and the severity rationale draw from it — that size-over-life therefore cannot be authored. It can, and it now is, in five shipped emitters.
  **The route.** `ScaleSpriteSize`'s `Scale Sprite Size Mode` is a **static switch**, and this ticket only ever considered its two named branches as endpoints. On the `Uniform` branch, `Uniform Scale Factor` is a plain float input — and a plain float input accepts a **dynamic input chain**. Assigning `Lerp_Float` there, with `RampInOut` driving its Alpha, produces a real animated size ramp with no curve data interface anywhere in it:
    `niagara.set_static_switch {assetPath, entryId, inputName:"Scale Sprite Size Mode", value:"Uniform"}`
    `niagara.set_module_input {assetPath, entryId, inputName:"Uniform Scale Factor", value:{dynamicInput:"/Niagara/Modules/DynamicInputs/Float/Lerp_Float.Lerp_Float"}}`
    then set the chain's `A`, `B` and the `RampInOut` alpha through the compound input names, compile, save.
  **Why this is trustworthy and not another dotted-path illusion.** The obvious objection is `B-niagara-module-input-dotted-subinput-silent-noop`: compound writes that land in a rapid-iteration parameter nothing reads. It does not apply here. The written values were verified in the emitter's `nir.txt` as **real graph override pins carrying links**, with **no** corresponding rapid-iteration entries — i.e. the graph reads them. That is the distinguishing evidence, and it is the check anyone re-testing this should repeat rather than trusting a success echo.
  **Applied to:** `E_Explosion_Shockwave` (0.09 -> 1.0 over life), `E_Explosion_DustRing` (0.35 -> 1.0), and the three `NS_Smoke_Grenade` emitters. All compiled, saved, re-added to their systems with handle parity, and confirmed on disk.
  **The finding that actually deserves this ticket's severity.** The same `Lerp_Float` + `RampInOut` chain was **already authored** on the muzzle smoke emitters, with real A/B values, and had never run — because their `Scale Sprite Size Mode` switch sat on `Uniform Curve`, which routes around the float input entirely and reads the unreachable DI instead. So the asset carried convincing evidence of a working ramp that the switch silently bypassed, and every readback of the chain's values looked correct. That silent bypass is a worse defect than the unreachable DI: the DI at least fails loudly with `DATA_INTERFACE_NOT_FOUND`, whereas this one shows an authored, plausible, entirely inert chain. A `niagara.validate` warning for "dynamic-input chain on an input the current static-switch branch does not read" would have caught it, and would catch the same class of dead configuration on any module.
  Found while fixing invisible dust and smoke on the FPS VFX package (UE 5.8, EAContentExamples58). Not a source read — evidence is the `nir.txt` override-pin dumps, the pre/post module lists, and the five emitters now shipping with working ramps. Severity left at High for the original reporter to re-judge; my own reading is that the impact half drops (a route exists) while the discoverability half rises (an authored chain can be silently inert).
- `#6-third-distinct-consequence-light-decay-cannot-be-authored-either` `OPEN` builder — **A third thing this defect blocks, found fixing the FPS muzzle and explosion lights.** `#1`-`#4` are about size-over-life and colour-over-life. This is light-over-life, and it fails the same way for the same reason.
  `NiagaraLightRendererProperties` has `bAlphaScalesBrightness`, which multiplies the emitted light by the particle's alpha — the natural way to make a muzzle flash or an explosion light *decay* rather than switch off. On `E_Explosion_Light` that alpha comes from the stock `ScaleColor` module's inline curve, `FloatFromCurve001.FloatCurve`. `niagara.set_curve_keys` returns `DATA_INTERFACE_NOT_FOUND` for it in **every** scope the verb accepts, because the DI lives on a graph pin rather than in any parameter store — exactly the shape `#1` describes.
  So the choice was: leave the light's brightness multiplied by a curve that cannot be read or authored, or turn `bAlphaScalesBrightness` off and lose the decay. We took the second, because a value you cannot inspect silently scaling a physical intensity is worse than a constant. The explosion light is now constant per particle, with a two-particle 0.06-0.40 s random lifetime standing in for a smooth falloff. That is a visible quality compromise in one of the six effects the project brief names, forced entirely by this verb gap.
  Worth noting for whoever scopes the fix: the workaround that rescued size-over-life (`#5`'s static-switch + dynamic-input route) does **not** transfer here. There is no static switch offering a non-curve branch for `ScaleColor`'s alpha, and the nested-dynamic-input route is itself a compile-breaking trap (`F-niagara-dynamic-input-nested-inputs` `#4`). For light decay specifically no alternative route has been found, which makes this the strongest case in the ticket for fixing the verb rather than routing around it — subject to the same caveat recorded in `#7`, that a node/pin-targeted `set_property` was never attempted.
  Evidence: `E_Explosion_Light` before/after property reads; `set_curve_keys` refusals across scopes; the emitter and `NS_Explosion` re-saved and verified on disk with the new constant colour in the system's own bytes.
- `#7-fourth-consequence-alpha-over-life-is-unreadable-which-forced-a-workaround-in-a-different-dimension` `OPEN` builder — **Alpha-over-life is unauthorable too, and that is what made a review defect hard to fix.** `ScaleColor`'s `Scale Alpha` is driven by a `FloatFromCurve` whose curve is a graph-local `NiagaraNodeInput` on the module stack node. `niagara.set_curve_keys` returns `DATA_INTERFACE_NOT_FOUND` for `Scale Alpha.FloatCurve001` in `updateRapidIteration`, `spawnRapidIteration` **and** `rendererBindings`; it appears in no parameter store; and neither `niagara.inspect {includeGraphs:true}` nor `asset.dump`'s `nir.txt` carries the curve keys. So the ramp is not merely unwritable through this verb — **it cannot be read either** through `inspect {includeGraphs:true}` or `nir.txt`, which means a caller cannot even discover the shape they are failing to change.
  Concrete consequence, from the VFX build. A critic scored `NS_Impact_Concrete` 3/10 with "the dust arrives so late and faint it is barely above background", and hypothesised concrete's own alpha/size ramp was at fault. A normalised NIR diff against `E_ImpWood_Dust` — which produces its cloud correctly — showed the two emitters are **structurally identical**: same modules, same entryIds, same `ScaleColor -> FloatFromCurve001` alpha ramp, same spawn time, same initial alpha. There was no ramp difference to fix. The real cause was that the ramp is driven by **normalised** age, so concrete's 1.6 s lifetime meant only ~2.5% of the ramp had elapsed at the 40 ms the effect is judged at.
  The fix had to be made in a different dimension entirely — cutting lifetime to 0.30/0.75 s so that more of the (unreadable, unwritable) ramp elapses in the first two frames, and raising spawn count to compensate for the shorter life. That works, but it is tuning one parameter to compensate for another that the API cannot touch, and nobody reading the asset later will see why the lifetime is what it is.
  This is now the **fourth** distinct thing this one gap blocks: size-over-life (`#1`-`#4`), colour-over-life (`#1`), light-decay-over-life (`#6`), and alpha-over-life (here). Only the size one has a known workaround (the static-switch route in `#5`). **Scope of the negative result, stated precisely:** what has been proven is that no route exists through the *documented* verbs — `set_curve_keys` refuses in all three emitter-scoped stores, and neither `inspect {includeGraphs:true}` nor `nir.txt` emits the keys. The DI is a graph-local `NiagaraNodeInput` on the module stack node, so a **node- or pin-targeted `niagara.set_property`** may still reach it; that was not tried. This is "no route found through the documented surface", not "no route exists", and the distinction is what a fix should key off — if the pin route works, the gap is discoverability and documentation rather than capability. A verb that could address a graph-local curve DI — or an `asset.dump` that at least *emitted* the curve keys so they could be reasoned about — would close all four.
  Evidence: `set_curve_keys` refusals across all three scopes; the normalised NIR diff of concrete vs wood dust; before/after lifetimes and spawn counts on `E_ImpConcrete_Dust`, re-added to `NS_Impact_Concrete` (5/5) and saved.
- `#8-read-gap-confirmed-on-a-system-asset-and-through-the-live-decompile-rpc` `OPEN` builder — **Narrowing `#7`'s negative result by one surface, and recording what the unreadable curve actually cost this time.** `#7` proves the keys are absent from `niagara.inspect {includeGraphs:true}` and from `asset.dump`'s `nir.txt`. Adding the third: **`niagara.decompile_nir` called live on a *system* asset emits them nowhere either.** On `/Game/FPS/VFX/NS_Smoke_Grenade` (UE 5.8, EAContentExamples58, port 27145, 2026-09-03) the 613 KB NIR payload contains 354 occurrences of `Curve` and zero of `Keys` or `InterpMode`. What it does publish is the complete wiring and nothing else:

  ```
  node `Scale Alpha.FloatCurve001` : NiagaraNodeInput
  set $FloatFromCurve001.FloatCurve = %NiagaraNodeInput_1.Input
  link `Scale Alpha.FloatCurve001`.Input -> `Map Set`_2.`FloatFromCurve001.FloatCurve`
  link `Float from Curve 001`.Value -> `Map Set`.`ScaleColor.Scale Alpha`
  call @FloatFromCurve (input DefaultCurve = dataInterface NiagaraDataInterfaceCurve)
  ```

  So the decompiler resolves the DI as a graph-local `NiagaraNodeInput`, names it, and links it — it simply never serialises the samples. That is the shape of the cheapest possible fix: the reader already reaches the object. `niagara.inspect {includeGraphs:true}` reports that pin as `defaultObject: ""` with `linkCount: 0`, which is actively misleading, since the NIR from the same asset shows the input node is linked and carries a real DI. And the `parameters` aspect lists `Constants.<Emitter>.FloatFromCurve001.Scale Curve = 1.0` — the scalar *multiplier* on the curve — while omitting the curve, so a caller sees a value that looks like the answer and is not.

  **What it cost.** Task was to fix "no dense core" on `NS_Smoke_Grenade`: the effect reads as a thin grey smudge at 400 ms. Two causes were measurable and were fixed (a `DepthFade` at 40 uu eroding the cloud against the wall it sits on, and `SpawnBurst_Instantaneous.Spawn Count = 1` on all three emitters). The third candidate — whether `ScaleColor`'s alpha ramp suppresses alpha early in life, which decides entirely whether a burst at t=0 is visible at the 120 ms the effect is judged at — **could not be checked at all**, so the fix ships with an unquantified caveat on exactly the frame the reviewer will look at. Writing a known curve to remove the ambiguity was rejected on principle: with no read path, an overwrite of an authored curve is a blind clobber of another agent's work, which the project's parallel-agent rules forbid. That is the practical shape of a read gap on a shared asset — it does not merely slow a caller down, it converts a safe edit into an unsafe one.

  **A read-only fix would close most of this ticket.** `#1`-`#7` want `set_curve_keys` to accept a module-input target. Worth noting for scoping: four of the seven consequences recorded here (`#2`'s "cannot see what the curve currently is", `#6`'s constant-light compromise, `#7`'s concrete-vs-wood diff, and this one) are blocked by the **read** half alone, and the read half is strictly easier — the decompiler already holds the object. Shipping the keys in `nir.txt` / `decompile_nir` / `inspect {includeGraphs:true}` before the write verb lands would unblock those four and make any later blind-write argument moot. Not source-confirmed: no read of the decompiler or the `set_curve_keys` handler; evidence is the three payloads above from the live editor.

- `#9-correction` `OPEN` VFX — **encounter `#5`'s "confirmed on disk" claim does not hold on this checkout, for the `NS_Smoke_Grenade` half.** `#5` states the `ScaleSpriteSize` work was "compiled, saved, re-added to their systems with handle parity, and confirmed on disk". Measured on this tree at 2026-09-03T04:40Z:

  ```
  NS_Smoke_Grenade.uasset       ScaleSpriteSize=0   RampInOut=0
  Emitters/E_Smoke_Core.uasset  ScaleSpriteSize=4   RampInOut=4
  Emitters/E_Smoke_Smoke.uasset ScaleSpriteSize=4   RampInOut=4
  Emitters/E_Smoke_Wisps.uasset ScaleSpriteSize=4   RampInOut=4
  ```

  The three emitter ASSETS carry the modules; the SYSTEM carries none of them, and the system's mtime is *later* than all three emitters', so this is not a save-ordering slip. The handles were never re-added. I re-derived this myself rather than relaying it: it was first reported by the Build 05 smoke fixer, and I byte-checked all four packages independently before writing this.

  This board is shared by several hosts, so the claim may well have been true where it was written — that is exactly why it is filed as a correction scoped to a named checkout rather than as a contradiction.

  **Visible consequence, now photographed.** `Docs/fps/evidence/vfx/b5_10_smoke_120ms_core.png`: with no size-over-life ramp the sprites are born at final size (90-150 uu) and hold it for their whole 2.5-4.5 s life, so at 120 ms the grenade is a wide low haze instead of the tight 55-70 cm knot the fixer's own model predicted. The smoke never billows. That is a second, independent quality defect riding on this ticket's blocked capability, and it is the first time the consequence has been captured in a frame rather than inferred from a stack read.

  **Practical trap for the next agent, from the same fixer:** the documented `remove_emitter` + `add_emitter` recovery re-copies from the emitter asset, so running it after editing a system's emitter copies in place DESTROYS those edits. On this asset that would have discarded six landed changes. There is currently no non-destructive route to re-add a missing handle mid-iteration.
- `#10-module-input-addressing-added-to-the-curve-verbs` `IN-REVIEW` developer — Source-confirmed the defect first (nine encounters, none of them a source read): `niagara.set_curve_keys` built `ENiagaraEditTargetKind::ParameterStore` and resolved through `Target.ParameterStore` only, so a module-pin DI was unreachable by construction rather than by a spelling mistake. Fixed by teaching `set_curve_keys` and `bind_curve_asset` the `entryId` + `inputName` form `niagara.set_module_input` already takes, over one shared module-input -> DI resolver (`NiagaraModuleInputDataInterface.h/.cpp`) rather than a parallel lookup, and adding `niagara.get_curve_keys` for the read half four of the encounters are blocked by. A default-valued input gets its own override DI seeded from the module script default before any write, so the shared module asset is never edited. Full detail, file list and the reviewer recipe are in the `## Fix` section above. NOT compiled and NOT test-run — a separate compile pass follows, and no live editor was touched.
