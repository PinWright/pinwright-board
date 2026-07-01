---
id: B-set-niagara-param-no-validation
title: "Component-side Niagara param setters (effect.set_niagara_parameter, niagara.modify_parameter) report success for parameters that don't exist on the system"
status: IN-REVIEW
severity: High
category: bug
tags: [effect, niagara, set-niagara-parameter, modify-parameter, silent-failure]
---

# Runtime Niagara param setters silently succeed for non-existent parameters

Both component-side runtime parameter setters share an identical silent-success
defect:

- `effect.set_niagara_parameter` (`Handlers/VFX/EffectHandler.cpp`) looks up the
  actor by label, finds its `UNiagaraComponent`, and — based on `parameterType`
  — calls one of `SetVariableFloat` / `SetVariableVec3` /
  `SetVariableLinearColor` / `SetVariableBool`. It sets `bApplied = true` as
  soon as the JSON `value` parses into the requested type, then returns
  `{"success":true,"applied":true, ...}`.
- `niagara.modify_parameter` (`Handlers/Niagara/NiagaraHandler.cpp`) is the twin:
  it resolves an `ANiagaraActor` by label, gets its `UNiagaraComponent`, and
  calls `SetFloatParameter` / `SetVectorParameter` / `SetColorParameter` /
  `SetBoolParameter` on it, setting `bSuccess = true` purely on JSON-value parse,
  returning `{"success":true, ...}`.

Neither handler checks whether the named parameter actually exists on the Niagara
system. Every `UNiagaraComponent::SetVariable*` / `SetFloatParameter`-style
setter forwards to `OverrideParameters.SetParameterValue(..., /*bAdd=*/true)`,
which writes into the component's override store for *any* name and **creates**
an entry for an unknown one — the write is a no-op against the system's real
parameters but no error is raised. So a typo'd name, a wrong `User.` prefix, or
a wrong case/type all return the same affirmative success as a real parameter —
with zero RPC-visible signal that nothing was tuned.

This is the same defect class already accepted on the board as
`B-input-trigger-modifier-stub-silent-success` (IN-REVIEW) and
`B-material-break-connections-named-pin-noop` (DONE): a handler returns
`SendSuccess` with an affirmative flag without verifying the mutation landed.

**Why it matters:**

- The realistic task that surfaced this ("tune three user-exposed parameters so
  the value is interpreted correctly") cannot distinguish a successful tune from
  a complete no-op. An agent that fat-fingers `User.SpawnRte` or omits the
  `User.` prefix gets `applied:true` and reports the fountain as tuned; the
  system runs with its defaults. The same is true via the documented
  `niagara.modify_parameter` workflow, so steering callers off `effect.*` toward
  `niagara.*` (as `docs/wiki-src/effect.md` does) does **not** save them.
- The sibling Niagara handlers already know how to validate a name against the
  system's exposed parameters —
  `System->GetExposedParameters().FindParameterVariable(...)` is used in
  `Handlers/Niagara/NiagaraGraphHandler.cpp` (lines ~450, ~462) and
  `Handlers/Niagara/NiagaraAdvancedEditHandler.cpp` (line ~483). So this is a
  missing validation, not an inherent engine limitation. `UNiagaraComponent`
  exposes `GetAsset()` and `GetOverrideParameters()` publicly, so the same
  validation is reachable from the runtime component path.

**Fix:** Add a shared validation helper
(`EditorAutomationNiagara::ComponentExposesParameter`) that resolves the
component's system via `NiComp->GetAsset()` and checks
`System->GetExposedParameters().FindParameterVariable(FNiagaraVariable(<typedef>, ParamName))`
for the requested `parameterType` (the `FNiagaraUserRedirectionParameterStore`
override resolves the `User.` prefix). Both `effect.set_niagara_parameter` and
`niagara.modify_parameter` call it before writing; if the variable is absent they
return `SendError("PARAMETER_NOT_FOUND", ...)` instead of an affirmative success.

## Repro

1. `effect.spawn_niagara({systemPath:
   "/Game/ExampleContent/Niagara/Simple/Simple_system.Simple_system",
   location:[0,0,200], name:"FountainPreview", autoDestroy:false})` → success.
2. `effect.activate_niagara({systemName:"FountainPreview", reset:true})` → `active:true`.
3. `effect.set_niagara_parameter({systemName:"FountainPreview",
   parameterName:"User.ThisParameterDefinitelyDoesNotExist",
   parameterType:"Float", value:999})`
   → returns `{"success":true,"applied":true,
   "parameterName":"User.ThisParameterDefinitelyDoesNotExist",
   "parameterType":"Float"}` — the parameter does not exist on
   `Simple_system`, yet the call reports it as applied.
4. Same with a no-prefix name and `parameterType:"Color"`:
   `parameterName:"CompletelyBogusNoPrefix"` → `applied:true` again.
5. `niagara.modify_parameter({actorName:"FountainPreview",
   parameterName:"User.AlsoBogus", parameterType:"Float", value:42})`
   → returns `{"success":true, ...}` — same silent no-op via the documented
   runtime workflow.

(The three real params from the task — `User.SpawnRate` Float / `User.Color`
Color / `User.Velocity` Vector — also return success, but they are
indistinguishable from the bogus names above, which is the bug.)

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode realism task ("tune user-exposed Niagara params at runtime") replayed live against `effect.set_niagara_parameter`. The three requested params (`User.SpawnRate`/`User.Color`/`User.Velocity`) returned `applied:true` as expected, but so did two deliberately bogus names (`User.ThisParameterDefinitelyDoesNotExist`, `CompletelyBogusNoPrefix`) — confirmed silent success-with-no-effect: the handler calls `UNiagaraComponent::SetVariable*` for any name and sets `applied:true` purely on JSON-value parse, never validating against `System->GetExposedParameters()`. Sibling handlers (NiagaraGraphHandler.cpp ~450/462, NiagaraAdvancedEditHandler.cpp ~483) already do this `FindParameterVariable` validation, so the fix is feasible. Same class as IN-REVIEW B-input-trigger-modifier-stub-silent-success / DONE B-material-break-connections-named-pin-noop. No existing board ticket references `set_niagara_parameter` (ripgrep clean across OPEN/DONE/WONTFIX).
- `#2-rescope-twin-setter` `IN-REVIEW` developer — Rescoped from one handler to both component-side runtime setters: adversarial review found `niagara.modify_parameter` (NiagaraHandler.cpp:444) has the IDENTICAL silent-add defect (`SetFloatParameter`/`SetVectorParameter`/`SetColorParameter`/`SetBoolParameter` set `bSuccess=true` on parse alone; in UE 5.7 these forward to `SetVariable*` → `OverrideParameters.SetParameterValue(..., bAdd=true)`, which silently creates unknown names). Fixing only `effect.set_niagara_parameter` would have left an identical no-op in the documented `niagara.modify_parameter` workflow. Title/body/Fix updated to cover both; severity High and category bug unchanged.
- `#3-validate-both-setters` `IN-REVIEW` developer — Added shared `EditorAutomationNiagara::ComponentExposesParameter(Component, ParamName, ParameterType)` (Handlers/Niagara/NiagaraInstanceUtils.h/.cpp) that maps the parameterType string onto a Niagara typedef (Vector accepts Vec3 or Position) and validates via `Component->GetAsset()->GetExposedParameters().FindParameterVariable(...)` — the redirection store resolves the `User.` prefix; fails closed on null component/asset/unknown type. Wired it into `effect.set_niagara_parameter` (Handlers/VFX/EffectHandler.cpp, new `bParameterExists` gate before the SetVariable* branches; unknown name now returns `PARAMETER_NOT_FOUND` instead of `applied:true`) and `niagara.modify_parameter` (Handlers/Niagara/NiagaraHandler.cpp, validates after the component resolve + an explicit invalid-type guard; unknown name returns `PARAMETER_NOT_FOUND`). Regression test: `Private/Tests/Niagara/TestNiagaraRuntimeParamValidation.cpp` spawns a real `ANiagaraActor` into the editor world with a seeded `User.SpawnRate` param and drives both handlers through the production dispatcher (`InvokeHandlerWithCapture`): bogus names (with and without `User.` prefix, Float/Color) must return `PARAMETER_NOT_FOUND`, real names must still succeed. The bogus-name assertions fail if either handler's validation is reverted. Not compiled/run here (later phase verifies).
