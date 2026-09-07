---
id: F-niagara-dynamic-input-nested-inputs
title: "niagara.set_module_input: recursive nested-input authoring on an assigned dynamic input"
status: OPEN
severity: Medium
category: feature
tags: [niagara, authoring, dynamic-input, parity-ue58]
encounters: 4
costly: 3
lastSeen: 2026-09-02T22:45:00+05:00
---

# niagara.set_module_input: recursive nested-input authoring on an assigned dynamic input

Split out from **F-niagara-dynamic-input-authoring** (which delivered assigning a dynamic input to a module input via `{ dynamicInput: "<path>" }`). That leaves the assigned dynamic input at its in-script default inputs. Many real dynamic inputs must have their OWN inputs set to be useful (a Curve-over-Life needs its curve; an Add/Multiply needs its addend/factor). This ticket adds recursive nested-input authoring:

```
value: { "dynamicInput": "<ScriptAssetPath>", "inputs": { "<inputName>": <literal | { dynamicInput, inputs }> } }
```

so a chain can be authored in one call.

Implementation note / why separate: setting a nested input requires the assigned dynamic-input node's authorable stack inputs (their names AND types) to resolve `inputName` and type the nested override pin. `NiagaraEdit::EnumerateScriptInputs` does NOT provide these — it surfaces only the script's parameter-map input node (observed: Add_Float enumerates a single `NewInput` of type `NiagaraParameterMap`, not its value inputs). The correct source is `FNiagaraStackGraphUtilities::GetStackFunctionInputs` (NIAGARAEDITOR_API) which needs an `FCompileConstantResolver` built from the target system/emitter + script usage. That machinery (and a nested-input regression test that discovers a real input name via the same resolver) is the work this ticket tracks.

Acceptance: assign a dynamic input with a nested input set (e.g. `{ dynamicInput: "Add_Float", inputs: { "<A>": 42.0 } }`); the call succeeds, the module input override pin is driven by the dynamic-input node, and the named nested input carries the set literal (or a further nested dynamic-input node). Depth-guard runaway/cyclic chains.

## Encounter 2026-08-27 — hit in real authoring; adds a failing call, a cheaper implementation path, and a severity argument

Observed 2026-08-27 on UE 5.8 in the EAContentExamples58 checkout, PinWright at
`8e76cad5`, while building the Atlantis level's VFX. Three things this ticket did
not previously record.

**1. The observed failure has an error code, and this ticket records no failing call
at all.** Assigning `{ dynamicInput: ... }` creates the node as designed, but the
node's own inputs never become authorable pins, and `niagara.set_module_input`
aimed at the **dynamic-input node's own `entryId`** — the natural thing to try next —
returns `INVALID_STACK`. That is not a "feature absent" response; it is the same
error a caller gets for a malformed module target, so a caller cannot tell the
capability is missing from the response. Worth wording the eventual refusal (or the
docs) so the two are distinguishable. The source acknowledgement this ticket is
named in is confirmed still present at
`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp:1495-1499`.

**2. A cheaper implementation path this ticket never considered.** The nested values
**do** land in the emitter rapid-iteration store, addressable as
`Constants.<Emitter>.<FunctionName>.<Input>`. So the feature is reachable today
through `niagara.set_parameter` with an emitter rapid-iteration scope — **no**
`GetStackFunctionInputs` and **no** `FCompileConstantResolver` machinery required.
That route is blocked only by `B-niagara-set-parameter-emitter-scope-unreachable`
(the emitter scopes demand an `emitter` argument the verb never declares, so they
answer `EMITTER_REQUIRED` without it and `UNKNOWN_PARAMS` with it). Fixing that
one-line schema gap may deliver most of this ticket's user-visible value for a
fraction of the scoped work. It would not cover *name/type discovery* of the nested
inputs — the caller would still have to know the input's name — but it does cover
setting them.

**3. The `Low` severity looks under-rated — recording the argument, not changing the
rating.** `#1` set `Low` on scheduling grounds ("tracked separately at lower
priority"), not on impact. In practice `RandomRangeFloat` and `RandomRangeVector2D`
are stuck at their 0–1 script defaults, which means **per-particle random ranges are
unavailable at all** — a basic requirement of almost any particle effect (this
session had to drive variation from `ScaleSpriteSizeBySpeed` / `ScaleColorBySpeed`
instead). And the documented escape hatch — "set a literal over the dynamic input
instead" — does not work either: `B-niagara-literal-over-linked-override-pin`
(filed from this session) shows that write is a silent no-op that reports success
and echoes the literal back. So the workaround that kept this at `Low` is gone. By
the board rubric this reads as a hard blocker with no workaround on a common path.
Leaving `severity` untouched — a reporter recommends, the ticket's author or a
triager decides.

## History
- `#5-bumped-by-cost` `OPEN` orchestrator — Severity Low -> Medium by cost. Costly encounters counted: #2 (Atlantis VFX — `RandomRangeFloat`/`RandomRangeVector2D` stuck at their 0-1 defaults left per-particle random ranges unavailable, and the documented escape hatch was itself a silent no-op, so the workaround that justified `Low` did not exist), #3 (FPS `NS_Impact_Concrete` — refuted #2's cheaper rapid-iteration route, then had to tear both dynamic inputs back out and rebuild speed and count variance out of unrelated modules), #4 (FPS `NS_Explosion` / `NS_Smoke_Grenade` — the write emitted invalid HLSL, ten particle scripts went to `NCS_Error`, both systems refused to activate, and a VFX critic scored them 0, failing the package review while four verbs reported healthy). Two independent streams, the Atlantis showcase VFX pass and the FPS VFX build. Reach argues the same way: the affected shape is an over-life ramp, which a sibling ticket recommends as its own workaround.
- `#1-split-from-dynamic-input` `OPEN` reporter — Split from F-niagara-dynamic-input-authoring during implementation: the flat assignment shipped, but recursive nested `inputs` needs resolver-based stack-input enumeration (`GetStackFunctionInputs` + `FCompileConstantResolver`) that `EnumerateScriptInputs` cannot supply, plus its own test, so it is tracked separately at lower priority.
- `#2-encounter-error-code-cheaper-path-severity` `OPEN` reporter — Additional evidence from real authoring (Atlantis level VFX; map as forcing function, see host `CLAUDE.md`), 2026-08-27, UE 5.8, PinWright at `8e76cad5` in the EAContentExamples58 checkout. Three new things over `#1`: (a) the observed failure now has a code — `set_module_input` against the dynamic-input node's own `entryId` returns `INVALID_STACK`, indistinguishable from a malformed module target, where this ticket previously recorded no failing call at all; (b) a cheaper implementation path — the nested values DO land in the emitter rapid-iteration store as `Constants.<Emitter>.<FunctionName>.<Input>`, so the feature is reachable via `niagara.set_parameter` with a rapid-iteration scope with no `GetStackFunctionInputs` / `FCompileConstantResolver` machinery, blocked today only by `B-niagara-set-parameter-emitter-scope-unreachable`'s missing `emitter` declaration (it would still not solve nested-input name/type discovery); (c) a severity argument — `#1`'s `Low` was set on scheduling grounds, but `RandomRangeFloat`/`RandomRangeVector2D` stuck at 0–1 defaults means per-particle random ranges are unavailable at all, and the documented escape hatch ("set a literal over the dynamic input") is itself a silent no-op per `B-niagara-literal-over-linked-override-pin`, so the workaround that justified `Low` does not exist. Recommending a re-rate; `severity` deliberately left unchanged. Source acknowledgement re-verified in this tree at `NiagaraEditHandler.cpp:1495-1499`. `encounters` 1→2, `lastSeen` refreshed.
- `#3-rapid-store-route-refuted` `OPEN` reporter — Third encounter, 2026-09-02, UE 5.8, EAContentExamples58 checkout, while authoring `/Game/FPS/VFX/NS_Impact_Concrete`. **This encounter refutes `#2`'s point (b).** The cheaper `niagara.set_parameter` route does **not** work for a dynamic input that PinWright itself added, because the rapid-iteration parameter is never created for it. Measured: on a `SimpleSpriteBurst` duplicate, `niagara.add_module` + `set_module_input {dynamicInput:"/Niagara/DynamicInputs/UniformRange/V2/RandomRangeInt"}` on `SpawnBurst_Instantaneous.Spawn Count`, and the same for `RandomRangeFloat` on `AddVelocityInCone.Velocity Strength`; then `niagara.compile {force:true,wait:true}` (`status:"completed"`) and `niagara.inspect {parametersOnly:true}`. Neither `Constants.<Emitter>.RandomRangeInt.Minimum` nor `…RandomRangeFloat.Minimum` appears in `spawnRapidIteration` or `updateRapidIteration` — the only dynamic-input entry present is the template's own pre-existing `Constants.<Emitter>.FloatFromCurve001.Scale Curve`, which is there because the template author touched it in the stack UI. The rapid-iteration entries for a dynamic input's leaf inputs are created by the **editor stack view** (`UNiagaraStackFunctionInput` refresh), not by the compiler, and no PinWright verb drives that path — so `set_parameter` answers `PARAMETER_NOT_FOUND` and `add_parameter` would be inventing a name the compiled script has no reason to bind. The `GetStackFunctionInputs` + `FCompileConstantResolver` machinery `#1` scoped is therefore still required; `#2`(b) should not be used to descope this ticket. Also re-confirmed `#2`(a): the dynamic-input node's own `entryId` answers `INVALID_STACK` (with or without `scriptUsage`), and `niagara.set_pin_default` on the same node answers `PIN_NOT_FOUND` for `Minimum` — correctly, since `niagara.graph.get` shows the function-call node carries only `Evaluation Type`/`Fixed Random Seed`/`InputMap`/`Override Seed`/`Randomness Mode`/`Recalculate Random Each Loop` and the output pin; the value inputs are not pins on it. Practical consequence in this session: both `RandomRangeInt` and `RandomRangeFloat` had to be torn out again (`set_module_input` with `breakExistingLink:true`), and per-particle speed variance was rebuilt from `Mass Mode: Random` + `Drag {Ignore Mass:false}` + the cone module's own `Velocity Falloff Away From Cone Axis`, while shot-to-shot count variance was rebuilt from a second `SpawnBurst_Instantaneous` with `Use Spawn Probability`. Both are workarounds for a specific effect, not a general substitute for a random range. Related new ticket filed the same session: `B-niagara-module-input-dotted-subinput-silent-noop` — the dotted-sub-input spelling a caller naturally reaches for next (`"Spawn Count.Minimum"`) returns affirmative success while writing an unread rapid parameter. `encounters` 2→3, `lastSeen` refreshed.
- `#4-not-low-this-emits-invalid-hlsl-and-ships-a-dead-asset` `OPEN` builder — **Severity argument, with the shader error.** This ticket is filed `Low` on the understanding that a nested dynamic input's inputs are merely *unreachable* — written somewhere nothing reads. On UE 5.8 / EAContentExamples58 they are worse than unreachable: **the write lands, the emitter generates invalid HLSL, and every script in it fails to compile.**
  Setting `RampIn`/`RampOut` on a `RampInOut` that was itself driving `Lerp_Float.Alpha`, which was driving `ScaleSpriteSize.Uniform Scale Factor`, produced:
```
/*0795*/ Context.Map.UniformScaleFactor.Lerp_Float.Alpha = RampInOut_Emitter_FuncOutput_Output;
/*0796*/ Context.Map.UniformScaleFactor.Lerp_Float.Alpha.RampInOut.RampIn = Constant55;
(796): error: Invalid swizzle / mask 'RampInOut'   (member access on a float)
LogNiagaraCompiler: Error: 1 errors encountered compiling Vector VM shaders
```
  Consequences measured, not inferred: ten particle scripts across `NS_Explosion` and `NS_Smoke_Grenade` went to `NCS_Error`; both systems refused to activate (`effect.activate_niagara` -> `active:false`); a VFX critic scored both **0** and the package failed its review with two of the six effects the project brief names simply absent. Removing the module from the five affected emitters restored `compile.valid:true` and every script to `NCS_UpToDate`, confirming the nested chain was the sole cause.
  Every available signal said this was fine: `set_module_input` returned success with a value echo, `niagara.compile {force,wait}` returned `status:"completed"`, and `niagara.validate level:"strict"` returned `valid:true, errors:[]` (see `B-niagara-validate-green-while-scripts-ncs-error` `#3`). The only way to see it was grepping the editor log for `errors encountered compiling Vector VM shaders`.
  **Ask, revised:** rather than "support nested dynamic input inputs" (the current feature request), the minimum fix is to **refuse the write**. The generated HLSL is unconditionally invalid for this shape, so `set_module_input` can reject it at the call with a typed error instead of returning success and shipping a dead asset. Supporting the feature properly is still worth doing; refusing it is worth doing first and is much cheaper.
  I would argue this is not `Low`. Impact is a silently dead asset that four verbs call healthy; reach is any caller building an over-life ramp, which is the single most common thing a VFX author wants and which the sibling ticket `B-niagara-set-curve-keys-unreachable-module-input-di` pushes callers toward as its workaround — that ticket's recommended route leads directly into this one. Leaving the field at `Low` for the original reporter to re-judge rather than changing another agent's severity.
