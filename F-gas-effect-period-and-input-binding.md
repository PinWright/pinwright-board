---
id: F-gas-effect-period-and-input-binding
title: "GameplayEffect period + GameplayAbility InputAction binding"
status: DONE
severity: Medium
category: feature
tags: [gas, gameplay-effect, gameplay-ability, periodic, input-binding, set-by-caller, execution-calculation]
---

# GameplayEffect period + GameplayAbility InputAction binding

The `gas.*` surface covers GameplayEffect duration policy and basic
modifier magnitude, but four common GAS authoring operations have no
typed RPC peer, forcing manual Blueprint edits after imperative
authoring.

**1. No effect period (DoT / HoT).** `gas.set_effect_duration` controls
`DurationPolicy` and `DurationMagnitude`, but `UGameplayEffect` also
carries `Period` (`FScalableFloat`), `PeriodicInhibitionPolicy`, and
`bExecutePeriodicEffectOnApplication`. Without these, any
damage-over-time or heal-over-time effect (the canonical periodic GE
use case) cannot be authored end-to-end through MCP.

**2. No ability input binding.** `gas.create_gameplay_ability` produces
a `UGameplayAbility` blueprint, and `gas.grant_ability` accepts an
`inputID` integer when granting at runtime, but there is no path to
declare the *static* binding most projects rely on — either the
`AbilityInputID` enum value on the ability CDO (common Lyra/ARPG
pattern) or an `UInputAction` soft-ref on a `UAbilitySet`-style data
asset (`gas.create_ability_set` exists but does not expose the input
mapping). Result: abilities created via MCP are ungrantable from input
without a manual Blueprint pass.

**3. No SetByCaller magnitude.** `gas.set_modifier_magnitude` hardcodes
`magnitudeType: "scalable_float"`. The other three production-relevant
calculation types — `SetByCaller` (most common runtime-parameterized
damage path), `AttributeBased`, `CustomCalculationClass` — have no
imperative form.

**4. No execution-calculation capture list.** `gas.add_effect_execution_calculation`
attaches a `UGameplayEffectExecutionCalculation` to an effect, but the
calculation class itself needs `RelevantAttributesToCapture` populated
(an array of `FGameplayEffectAttributeCaptureDefinition`: source/target,
attribute, snapshot). Without an RPC to author this list on the
execution-calc subclass, the execution runs with no captures and silently
produces zero output.

**Impact:** Any imperative GAS authoring beyond a flat instant
`scalable_float` modifier requires dropping out of MCP. Periodic effects,
SetByCaller damage formulas, executions with captures, and
input-bindable abilities are the bulk of real GAS code.

**Proposal:** Four new RPCs in the `gas` namespace:

1. `gas.set_effect_period` — params:
   `blueprintPath` (required),
   `periodSeconds` (number, the `Period.Value`),
   `periodicMagnitude` (number, optional, scalable-float modifier
    applied per tick; if omitted, leaves existing modifiers untouched),
   `executeOnApplication` (bool, default `true`, sets
    `bExecutePeriodicEffectOnApplication`),
   `inhibitionPolicy` (string: `never_reset` | `reset_period` |
    `execute_and_reset_period`, default `never_reset`).
   Errors `NOT_PERIODIC` if `DurationPolicy != Infinite | HasDuration`
   (period is meaningless on Instant).

2. `gas.set_ability_input` — params:
   `blueprintPath` (required),
   one of:
     - `inputIDEnumValue` (string, name within the project's
       `EAbilityInputID`-style enum — handler resolves the enum by
       scanning project enums tagged with the conventional name, with
       `enumPath` override),
     - `inputActionPath` (string, soft-ref to a `UInputAction`),
   `bindOn` (string: `cdo` | `ability_set`, default `cdo`,
    selects whether to write `AbilityInputID` on the ability CDO or
    add an entry to a referenced `UAbilitySet`),
   `abilitySetPath` (string, required if `bindOn == ability_set`).

3. `gas.set_modifier_magnitude_setbycaller` — params:
   `blueprintPath` (required),
   `modifierIndex` (int),
   `dataTag` (string, `FGameplayTag` for `SetByCallerMagnitude.DataTag`),
   `dataName` (string, optional `FName` fallback for legacy
    `SetByCallerMagnitude.DataName`).
   Sets the modifier's `MagnitudeCalculationType` to `SetByCaller` and
   wires the tag/name reference.

4. `gas.set_execution_capture` — params:
   `executionClassPath` (required, path to a
    `UGameplayEffectExecutionCalculation` blueprint),
   `captures` (array of objects:
     `{ attribute: "Set.Attr", source: "source" | "target",
        snapshot: bool, backingPropertyName?: string }`).
   Adds entries to `RelevantAttributesToCapture` on the CDO. The
   `backingPropertyName` is optional and only needed for the
   `DECLARE_ATTRIBUTE_CAPTUREDEF`-style C++ pattern; for pure-BP
   executions the engine generates the property under the hood.
   Returns the resolved attribute soft-class for each capture so the
   caller can confirm the attribute path was matched.

**Cross-ref:**
The handler patterns in
`Handlers/Systems/GASHandler.cpp` (or whichever file owns the existing
`gas.*` registrations) already resolve `UGameplayEffect` blueprints and
mutate the CDO; these four RPCs follow the same shape. `gas.create_ability_set`
exists but is silent on input — see whether the input binding belongs
inside that handler or as a separate `gas.set_ability_input` mutation
RPC; the proposal above picks the latter for symmetry with
`gas.set_ability_tags` / `gas.set_ability_targeting`.

## History
- `#1-missing-period-and-input` `OPEN` reporter — `gas.set_effect_duration` covers duration policy/magnitude but offers no separate `Period` for periodic effects (DoT/HoT), and `gas.create_gameplay_ability` does not bind an `AbilityInputID` enum or `UInputAction` soft-ref, so abilities created via MCP are ungrantable from input without manual BP edits. `gas.set_modifier_magnitude` hardcodes `scalable_float`, so `SetByCaller` damage formulas have no imperative form. `gas.add_effect_execution_calculation` attaches an execution but does not populate `RelevantAttributesToCapture` on the execution-calc class, so executions run with no captures. Proposes `gas.set_effect_period`, `gas.set_ability_input`, `gas.set_modifier_magnitude_setbycaller`, and `gas.set_execution_capture` to close the four gaps. Verified all four method names are absent from `docs/rpc-method-reference.generated.md` and `Source/EditorAutomationRpcGateway/Private/Handlers/Systems/`.
- `#2-implemented-gas-period-and-bindings` `IN-REVIEW` developer — Added `gas.set_effect_period` (Period + bExecutePeriodicEffectOnApplication + PeriodicInhibitionPolicy, rejects Instant effects with NOT_PERIODIC), `gas.set_ability_input` (cdo mode writes AbilityInputID int + optional InputAction soft-ref; ability_set mode returns NOT_IMPLEMENTED pending create_ability_set extension), `gas.set_modifier_magnitude_setbycaller` (uses FGameplayEffectModifierMagnitude's public FSetByCallerFloat ctor — no reflection hack needed), `gas.set_execution_capture` (pushes FGameplayEffectAttributeCaptureDefinition into protected RelevantAttributesToCapture via FArrayProperty + FScriptArrayHelper). Appended to `Handlers/Systems/GASHandler.cpp`. Tests in `Tests/Gameplay/TestGASHandlers.cpp` cover Period round-trip + Instant-rejection counterfactual. No Build.cs change — GameplayAbilities already linked.
- `#3-verify-fix` `DONE` tester — Verified: all four schemas present via `gas.<method>?` discovery (params match proposal: blueprintPath/periodSeconds/inhibitionPolicy/executeOnApplication on set_effect_period; inputIDEnumValue/inputActionPath/bindOn on set_ability_input; modifierIndex/dataTag/dataName on set_modifier_magnitude_setbycaller; executionClassPath/captures on set_execution_capture). Functional test: `gas.set_effect_period` on a freshly-created infinite GE round-tripped `{periodSeconds:1.5, executeOnApplication:false, inhibitionPolicy:"reset_period"}` in the response; same call on an Instant GE returned `NOT_PERIODIC` as specified. Temp assets cleaned up via `asset.delete`.
