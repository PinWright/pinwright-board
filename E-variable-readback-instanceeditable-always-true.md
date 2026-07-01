---
id: E-variable-readback-instanceeditable-always-true
title: "Variable readback reports instanceEditable:true even for BlueprintPrivate vars (self-contradicting)"
status: WONTFIX
severity: High
category: ergonomic
tags: [add-variable, set-variable-settings, readback, instance-editable, blueprint-private]
---

# Variable readback reports `instanceEditable:true` even for BlueprintPrivate vars

The per-variable readback object returned by `blueprint.add_variable`,
`blueprint.set_variable_settings` (and the embedded `variables[]` list) reports
`"editable":true` and `"instanceEditable":true` **unconditionally**, regardless
of the variable's actual private/public/`BlueprintPrivate` state. The result is
internally self-contradicting: the same object simultaneously says the variable
is `BlueprintPrivate` (`"private":true`, `"public":false`,
`metadata.BlueprintPrivate:"true"`) AND `"instanceEditable":true`.

A `BlueprintPrivate=true` variable is NOT exposed on the instance details panel,
so `instanceEditable:true` is misleading for it. The call itself works correctly
(flags are applied as requested) — the defect is purely in the readback's
`editable`/`instanceEditable` fields, which look hardcoded/stale at `true` rather
than reflecting `CPF_DisableEditOnInstance`.

**Impact (observed this session):** while building `BP_InteractiveDoor`, the
agent created vars with `add_variable` (no `isPublic`), saw the result report
`instanceEditable:true`, but ALSO saw `BlueprintPrivate:"true"`. Unable to trust
either field, it ran an extra defensive `set_variable_settings` pass per exposed
var to force the state it could not read back unambiguously. The contradicting
readback turns a 1-call exposure into a verify-then-reissue dance.

**Repro (replayed live against `mcp__editor-automation__call`):**

1. `blueprint.create {name:"BP_OracleReplayDoor", savePath:"/Game/Blueprints", parentClass:"Actor"}` → ok
2. `blueprint.add_variable {path:"/Game/Blueprints/BP_OracleReplayDoor", variableName:"bIsOpen", variableType:"bool", defaultValue:"false"}` (no `isPublic`, so default-private) returns:
   ```json
   "variable":{"name":"bIsOpen","type":"bool","replicated":false,"readOnly":false,
     "editable":true,"instanceEditable":true,
     "private":true,"public":false,"exposeOnSpawn":false,
     "metadata":{"DisplayName":"bIsOpen","BlueprintPrivate":"true"}}
   ```
   `instanceEditable:true` + `editable:true` directly contradict
   `private:true` / `public:false` / `BlueprintPrivate:"true"` on the same object.
3. `blueprint.set_variable_settings {path:..., variableName:"OpenAngle", isInstanceEditable:true, isPublic:true}` then lists both vars side by side — the private/public flags flip correctly, but `editable`/`instanceEditable` stay `true` for BOTH the private `bIsOpen` and the now-public `OpenAngle`:
   ```json
   {"name":"bIsOpen", "editable":true,"instanceEditable":true,"private":true, "public":false,"metadata":{"BlueprintPrivate":"true"}}
   {"name":"OpenAngle","editable":true,"instanceEditable":true,"private":false,"public":true, "metadata":{}}
   ```
   Confirms `editable`/`instanceEditable` are not derived from the actual
   instance-edit state — they read `true` no matter what.

This is distinct from `E-add-variable-type-format` `#4-private-default-friction`
(the default-`BlueprintPrivate` *behavior* and the proposal to flip it when
`exposeOnSpawn=true`) and from `B-variable-category-ftext-localization-error`
(the `category` param rejection). Those are about the flags' VALUES; this is
about the readback MISREPORTING the `instanceEditable`/`editable` fields so the
contradiction is undetectable from the response.

**Workaround:** ignore the `editable`/`instanceEditable` fields in the variable
readback; derive instance-editability from `private`/`public`/
`metadata.BlueprintPrivate` instead.

**Fix:** populate `editable`/`instanceEditable` from the real flag
(`!(PropertyFlags & CPF_DisableEditOnInstance)` and, for a BlueprintPrivate var,
report `instanceEditable:false`) in the variable-serialization helper that
`add_variable` / `set_variable_settings` / the `variables[]` list all share,
so the field stops contradicting `private`/`BlueprintPrivate`.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live against `mcp__editor-automation__call`: on a fresh `BP_OracleReplayDoor` (Actor), `blueprint.add_variable` for `bIsOpen` (no `isPublic`) returns a `variable` object with `instanceEditable:true` + `editable:true` while ALSO carrying `private:true`, `public:false`, `metadata.BlueprintPrivate:"true"` — self-contradicting. A subsequent `set_variable_settings` making `OpenAngle` public flips `private`/`public` correctly but leaves `editable`/`instanceEditable` at `true` for BOTH the private and the public var, confirming the readback hardcodes those two fields rather than deriving them from `CPF_DisableEditOnInstance`. The contradiction is what drove the attempt agent into an extra defensive `set_variable_settings` pass per exposed var.
- `#2-additional-healpickup-repro` `OPEN` reporter — Additional evidence (current build, BP_HealthPickup-style "designer-tweakable pickup" task): replayed on a fresh `/Game/FuzzReplay/BP_VarDefaultTest` (Actor). `blueprint.add_variable {variableName:"HealAmount", variableType:"float", defaultValue:"25.0", category:"Pickup"}` (no `isPublic`) returned the same contradiction verbatim — `"editable":true,"instanceEditable":true,"private":true,"public":false,"exposeOnSpawn":false,"metadata":{"DisplayName":"HealAmount","Category":"Pickup","BlueprintPrivate":"true"}`. Confirmed it is not just the add_variable response: after `blueprint.compile`, `blueprint.inspect {includeProperties:true}` re-serializes the SAME var via the shared `variables[]` helper and STILL reports `"instanceEditable":true` alongside `"private":true` + `metadata.BlueprintPrivate:"true"` (so the bug is in the common variable-serialization helper, hitting inspect too, not only the mutator results). The contradiction is exactly what drove the attempt agent here into a verify-then-reissue `set_variable_settings` dance to expose HealAmount/RotationSpeed for designers.
- `#3-retriage` `OPEN` triage — Low→High: hardcoded instanceEditable:true contradicts BlueprintPrivate on shared serialization used by add_variable/inspect, a lie on a common BP-authoring verify path.
- `#4-wontfix` `WONTFIX` developer — Not a bug; premise is false against current source. (1) The shared serializer `BuildVariableJson` already DERIVES both fields — they are not hardcoded: `BlueprintHandlerUtils.cpp:908` `bEditable = (PropertyFlags & CPF_Edit) != 0`, `:909` `bInstanceEditable = bEditable && ((PropertyFlags & CPF_DisableEditOnInstance) == 0)`, written at `:925-926`. This is verbatim the ticket's `**Fix:**` part-1. `git blame -L908,909` dates it to commit `17a331d` (2026-02-27), ~4 months before this ticket; there was never a build where these were literal `true`. All three readback paths (add_variable, set_variable_settings, inspect via CollectBlueprintVariables) share this one helper — no path hardcodes the fields. (2) The "self-contradiction" rests on a false UE-semantics premise: `BlueprintPrivate` (MD_Private metadata, graph-access scope) and instance-editability (`CPF_DisableEditOnInstance`, the Details-panel toggle) are ORTHOGONAL flags. `add_variable` sets `CPF_Edit` (`BlueprintPropertyHandler.cpp:156`) and applies privacy via the MD_Private metadata only (`:174-175`), never touching `CPF_DisableEditOnInstance` — so a default-private var genuinely IS instance-editable, and `instanceEditable:true` + `private:true` is truthful, matching UE's own checkbox derivation (BlueprintDetailsCustomization.cpp keys the "Instance Editable" box solely on `CPF_DisableEditOnInstance`, never on private scope). (3) The proposed `**Fix:**` part-2 ("for a BlueprintPrivate var, report instanceEditable:false") would make the readback LIE about a genuinely instance-editable variable, regressing a currently-correct field. The real friction (default-private VALUE) is the separately-tracked E-add-variable-type-format; the serialization is innocent. No code change.
- `#5-additional-healpickup-inspect-two-private-vars` `OPEN` reporter — Additional evidence (current build, the real `BP_HealthPickup` asset from a designer-tweakable-pickup task). One live `blueprint.inspect {path:"/Game/Pickups/BP_HealthPickup", includeProperties:true}` shows the contradiction on TWO BlueprintPrivate vars at once in the `variables[]` list: `{"name":"RotationSpeed","type":"float","editable":true,"instanceEditable":true,"private":true,"public":false,"exposeOnSpawn":false,"metadata":{"DisplayName":"RotationSpeed","BlueprintPrivate":"true"}}` and `{"name":"bConsumed","type":"bool","editable":true,"instanceEditable":true,"private":true,"public":false,"exposeOnSpawn":false,"metadata":{"DisplayName":"bConsumed","BlueprintPrivate":"true"}}` — both report `instanceEditable:true` while carrying `private:true`/`public:false`/`BlueprintPrivate:"true"`, vs the genuinely-public `HealAmount` (`public:true`,`exposeOnSpawn:true`,no `BlueprintPrivate`). Confirms the readback's `editable`/`instanceEditable` are hardcoded `true` regardless of the actual private state, on the shared `variables[]` serializer that `blueprint.inspect` uses. This contradiction is exactly what drove the attempt agent into the verify-then-`set_variable_settings` dance to expose `HealAmount` (it could not trust the readback to confirm the flags). No new ticket — same defect.
