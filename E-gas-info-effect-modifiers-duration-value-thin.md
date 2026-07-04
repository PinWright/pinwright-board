---
id: E-gas-info-effect-modifiers-duration-value-thin
title: "gas.get_gas_info doesn't echo per-modifier op/magnitude/attribute or the duration value (GameplayEffect), nor the attribute list (AttributeSet) — widen both branches"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [gas, get_gas_info, gameplay-effect, attribute-set, modifier, duration, readback, inspect-after-mutate, periodic, set_effect_period]
encounters: 2
lastSeen: 2026-07-04T17:40:47+03:00
---

# `gas.get_gas_info`'s GameplayEffect / AttributeSet branches are too thin to confirm a modifier/duration/attribute edit

This is the **GameplayEffect + AttributeSet sibling** of the `get_gas_info`
readback-thinness family — `E-gas-info-skips-asc-owner-actor` (IN-REVIEW, widened
the GE branch with executions/tags + added the ASC-owner branch) and
`E-gas-info-ability-omits-cooldown-cost-tags` (IN-REVIEW, widened the
GameplayAbility branch with cooldown/cost/abilityTags). Those two explicitly left
the **GameplayEffect per-modifier / duration-value** and **AttributeSet
attribute-list** scope to this ticket. Resolved the same way they were: widen the
branch at the source so the documented readback can confirm the write, not by a
docs-only `asset.dump` fallback (the approach the audio sibling
`E-audio-get-info-soundclass-mix-readback-thin` was explicitly reworded *away*
from in favor of a live reader).

## GameplayEffect branch (`GASHandler.cpp:2778-2821`, `Systems/` subdir)

The GameplayEffect branch emits `gasType`, `durationPolicy` (a raw enum int,
`static_cast<int32>(EffectCDO->DurationPolicy)`), `stackingType`, `modifierCount`
(`EffectCDO->Modifiers.Num()`), `cueCount`, `executionCount`/`executionClasses`,
and `grantedTags`. It does **not** emit:

- **per-modifier detail** — only `modifierCount`, never each modifier's
  `ModifierOp` / `ModifierMagnitude` / `Attribute`. An agent that adds two
  modifiers and bumps one with `set_modifier_magnitude` can read back *that there
  are two* but not *which op, what magnitude, or which attribute each one carries*.
- **the duration value** — only the `durationPolicy` enum (`2` = HasDuration),
  never the `DurationMagnitude` ScalableFloat that `gas.set_effect_duration`
  writes (`GASHandler.cpp:1812`,
  `EffectCDO->DurationMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(Duration))`).
  So `get_gas_info` confirms the policy *is* has-duration but not that the
  duration *is 8s*.
- **a readable `durationPolicy`** — the int forces the caller to cross-reference
  `EGameplayEffectDurationType` in the engine header to decode `2`=HasDuration.

So a canonical GameplayEffect authoring task — create a has-duration GE, set 8s,
add two additive modifiers (+25, -10), bump index 0 to +30 — has its required
final readback ("confirm the duration policy **and** that both modifiers are
present with the right operations and magnitudes") only **half**-satisfied: it
confirms `durationPolicy:2` and `modifierCount:2`, but not the 8s value, the
per-modifier ops, or the magnitudes.

Note the per-modifier `attribute` name is readable **now**:
`F-gas-modifier-no-attribute-binding` landed — `gas.add_effect_modifier` accepts
an `attribute` param and sets `Modifier.Attribute` (`GASHandler.cpp:1836-1838`,
`:1904`), and `gas.set_modifier_attribute` (`:1963`) rebinds it — so the readback
can echo each modifier's bound attribute today (empty when unbound).

## AttributeSet branch (`GASHandler.cpp:2822-2825`)

The AttributeSet branch emits only `gasType:"AttributeSet"` and **never
enumerates the attributes**. An agent that authors an AttributeSet with
`gas.add_attribute` (×N, each carrying a default/base value) and is then asked to
"inspect the finished AttributeSet to confirm all three attributes and their
default values landed correctly" cannot satisfy that readback — there is no
`attributes` list at all, so neither names nor defaults are confirmable. Each
attribute is an `FGameplayAttributeData` `FStructProperty` on the generated class;
the `BaseValue` numeric subproperty holds the default `gas.set_attribute_base_value`
writes.

## Fix

Widen both branches of `gas.get_gas_info` at the source
(`Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`), right where the two
IN-REVIEW siblings already extend the same `else if` chain:

- **GameplayEffect branch:** add a `durationMagnitude` scalar read off
  `EffectCDO->DurationMagnitude.GetStaticMagnitudeIfPossible(1.f, Out)`; add a
  readable `durationPolicyName` string ("Instant"/"Infinite"/"HasDuration")
  alongside the existing raw int; and add a `modifiers` array, one entry per
  `EffectCDO->Modifiers`, carrying `operation` (a readable op string via
  `EGameplayModOpToString`), `magnitude` (the static scalar where derivable),
  `magnitudeType` (the `EGameplayEffectMagnitudeCalculation`), and `attribute`
  (the bound `FGameplayAttribute` name, empty when unbound).
- **AttributeSet branch:** add an `attributes` array, one entry per
  `FGameplayAttributeData` `FStructProperty` on the generated class, carrying
  `name` and `baseValue` (read off the `FGameplayAttributeData::BaseValue`
  subproperty on the CDO).

This closes the gap at the source so the documented GAS readback confirms a
`set_effect_duration` / `add_effect_modifier` / `set_modifier_magnitude` /
`add_attribute` run, and `modifierCount` becomes an at-a-glance summary alongside
the detail. Also add a short `## Inspect-after-mutate` note to
`docs/wiki-src/gas.md` describing the widened readback fields.

## Evidence (this task)

Focus `gas.add_effect_modifier`. Story: author `GE_BerserkersRage`
(`/Game/Abilities/Effects`), has_duration 8s, additive +25 (AttackPower) and
additive -10 (Armor), then `set_modifier_magnitude index 0 -> 30`, then **read the
finished effect back to confirm the duration policy and both modifiers'
operations and magnitudes**. The build ran clean (8 calls, all ok) but the final
readback could not be satisfied by `get_gas_info`: it returned `durationPolicy=2`,
`modifierCount=2` only, forcing an `asset.dump` -> `properties.json` pivot to
recover `modifier[0] ModifierOp=AddBase mag=30`, `modifier[1] ModifierOp=AddBase
mag=-10`, and `DurationMagnitude ScalableFloat=8` — exactly the three values the
story's step 6 readback named.

Replayed AttributeSet evidence: after `gas.create_attribute_set` + `gas.add_attribute`
×3 (Health=100, Mana=50, Stamina=75), `gas.get_gas_info` returns
`{...,"gasType":"AttributeSet"}` with no `attributes` list — the required "confirm
all three attributes and their default values" readback is unsatisfiable.

**Workaround (until this lands):** confirm per-modifier
`ModifierOp`/`ModifierMagnitude`/`Attribute`, the `DurationMagnitude` value, and
the AttributeSet attribute defaults via `asset.dump { assetPath }` ->
`properties.json`, not `gas.get_gas_info`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `gas.add_effect_modifier`-focused GameplayEffect build (GE_BerserkersRage: has_duration 8s, additive +25 / -10, set_modifier_magnitude idx0->30; outcome gap, judge filed `F-gas-modifier-no-attribute-binding` for the missing attribute param). Distinct PROCESS angle: `gas.get_gas_info`'s GameplayEffect branch emits only `modifierCount` (a count, never per-modifier `ModifierOp`/`ModifierMagnitude`/`Attribute`) and only `durationPolicy` (the enum, never the `DurationMagnitude` value written by `set_effect_duration`), so the story's required step-6 readback ("confirm the duration policy AND both modifiers' operations and magnitudes") was only half-satisfiable — the agent pivoted to `asset.dump` -> `properties.json` to recover modifier[0]=AddBase/30, modifier[1]=AddBase/-10, duration=8. Same readback-thinness shape as the audio sibling. Distinct from `E-gas-info-skips-asc-owner-actor` (whose #2/#3 widened the same branch only for executions/tags + the ASC-owner branch, NOT per-modifier or duration value) and from the write-side `F-gas-modifier-no-attribute-binding`.
- `#2-additional-attributeset-and-ge-replay` `OPEN` reporter — Additional evidence (seed `gas.get_gas_info`, realistic ARPG GAS-scaffold task). Two replayed-live confirmations. (a) Broadens this ticket to the **AttributeSet** branch: after `gas.create_attribute_set` + `gas.add_attribute` ×3 (Health=100, Mana=50, Stamina=75), the task's required readback "inspect the finished AttributeSet to confirm all three attributes and their default values landed correctly" is UNsatisfiable — `gas.get_gas_info` returns `{...,"gasType":"AttributeSet"}` (no `attributes` list, no per-attribute name, no default value). (b) Re-confirms the GameplayEffect half live: `gas.get_gas_info {assetPath:"/Game/GAS/Effects/GE_HealthRegen"}` (has_duration, set_effect_duration 10s, additive +5 Health) -> `{...,"gasType":"GameplayEffect","durationPolicy":2,"stackingType":0,"modifierCount":1,"cueCount":0,"executionCount":0,"executionClasses":[],"grantedTags":[]}` — `durationPolicy` is a raw enum int (caller had to cross-reference `EGameplayEffectDurationType` to decode `2`=HasDuration), and there is no `durationMagnitude` (10) nor any per-modifier op/magnitude/attribute (only `modifierCount:1`). Reinforces the proposed source-widening (a `modifiers` array + `durationMagnitude` scalar), surfacing `durationPolicy` as a readable string, and widening the AttributeSet branch with an `attributes` array (name + default/base value).
- `#3-rescope-to-source-widening` `IN-REVIEW` developer — Reworded from a docs-only `asset.dump` fallback to **source-widening**, matching the two IN-REVIEW `get_gas_info` siblings (the audio sibling was likewise reworded away from the docs fallback). Corrected stale citations (file moved to `Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`; GE branch `:2778-2821`, AttributeSet branch `:2822-2825`, `set_effect_duration` write `:1812`) and noted `F-gas-modifier-no-attribute-binding` has LANDED, so the per-modifier `attribute` name is readable now (not conditional). Implemented: (1) GameplayEffect branch — added a `durationMagnitude` scalar (`GetStaticMagnitudeIfPossible`), a readable `durationPolicyName` string, and a `modifiers` array (`operation` via `EGameplayModOpToString`, `magnitude`, `magnitudeType`, `attribute`); (2) AttributeSet branch — added an `attributes` array (per `FGameplayAttributeData` `FStructProperty`: `name` + `baseValue`). Helpers: `DurationPolicyToString`, `MagnitudeCalcToString`, `CollectEffectModifiers`, `CollectAttributeSetAttributes` in GASHandler.cpp. Docs: added a `## Inspect-after-mutate` section to `docs/wiki-src/gas.md`. Files: `Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`, `Docs/wiki-src/gas.md`. Test: `PinWright.gas.get_gas_info.EffectModifiersDurationAndAttributeSetAttributes` in `Source/PinWright/Private/Tests/Gameplay/TestGASHandlers.cpp` — builds an AttributeSet (AttackPower=12), a has_duration 8s GE with two additive modifiers (+25 AttackPower, -10) then bumps idx0 to 30, and asserts get_gas_info reads back `durationMagnitude≈8`, `durationPolicyName:"HasDuration"`, a 2-entry `modifiers` array with `operation:"Add (Base)"`/magnitudes 30 and -10 and the bound `attribute:"AttackPower"` on idx0, and the AttributeSet `attributes` array with `name:"AttackPower"`/`baseValue≈12`. Reverting any emit fails the matching assertion.
- `#4-additional-period-and-stacklimit` `OPEN` reporter — Additional evidence (seed `gas.set_effect_duration`; culprit `gas.get_gas_info`; realistic "GE_Burning" DoT authoring task). Broadens the GameplayEffect-branch widening to **two more write-side fields the `#3` fix did not add**, replayed live. After authoring `/Game/GAS/Effects/GE_Burning_Replay` (has_duration 8s, additive -5 modifier, `gas.set_effect_period {periodSeconds:1.5, executeOnApplication:true, inhibitionPolicy:"reset_period"}`, `gas.set_effect_stacking {stackingType:"aggregate_by_source", stackLimit:3}`), `gas.get_gas_info` returned `{...,"durationPolicyName":"HasDuration","durationMagnitude":8,"stackingType":1,"modifierCount":1,"modifiers":[{"operation":"AddBase","magnitude":-5,"magnitudeType":"ScalableFloat","attribute":""}],...}` — i.e. the `#3` widening is live (durationMagnitude/durationPolicyName/modifiers all present), but the readback has **no `period` field** (the `EffectCDO->Period` ScalableFloat that `gas.set_effect_period` writes at `GASHandler.cpp:3488`, `EffectCDO->Period = FScalableFloat(PeriodSeconds)`) and **no `stackLimit` field** (only `stackingType` is read back; the `EffectCDO->StackLimitCount` that `gas.set_effect_stacking` writes at `:2338` is never emitted — the GE branch emits `stackingType` at `:2944` only). So the task's named final step — "confirm the duration policy, duration seconds, **period**, modifier count/magnitude, **stacking** [limit], and granted tags all match what we set" — is satisfiable for everything EXCEPT period (1.5s) and stack limit (3), which could only be confirmed from each set-call's own echoed response (`periodSeconds`/`stackLimit`), not the final round-trip readback. Fix extends the same `#3` GameplayEffect-branch widening (`GASHandler.cpp` GE branch ~`:2917-2980`): emit a `period` scalar (`EffectCDO->Period.GetValueChecked()`/`GetStaticMagnitudeIfPossible`, plus optionally `executeOnApplication` / `inhibitionPolicy`) and a `stackLimit` int (`EffectCDO->StackLimitCount`) right next to the existing `stackingType` stamp — and extend the `#3` regression test to assert both round-trip.
- `#5-additional-executeonapplication-invisible-in-dump-too` `IN-REVIEW` reporter — Additional evidence + a sharpening angle on `#4`'s period-omission (process-audit; focus `gas.set_effect_period`, culprit `gas.get_gas_info`; infinite regeneration-buff task, GE_Regeneration period=2s executeOnApplication=true periodicMag=5). Re-confirms live that `gas.get_gas_info` emits no periodic fields (no `period`, no `executeOnApplication`, no `inhibitionPolicy`) — it returned only `durationPolicyName Infinite / modifierCount 1 / AddBase / Health`. The sharpening: unlike `period` (which the `asset.dump` -> `properties.json` fallback DOES recover — `Period.Value==2` is a non-default write and appears in the dump), `executeOnApplication` is invisible in the `asset.dump` fallback too, because the dump omits any property equal to the parent-CDO default and the engine default `bExecutePeriodicEffectOnApplication` is already `true`. So the one workaround that saves `period` cannot confirm `executeOnApplication==true` at all: the agent had to grep the engine source for the default (`GameplayEffect.cpp` default=true), then run a destructive false-write differential — `set_effect_period executeOnApplication=false` -> `asset.dump` (now `bExecute:false` appears) -> `set_effect_period executeOnApplication=true` -> `asset.save` to restore — ~9 calls plus a real asset mutation just to observe one boolean persisted. This upgrades `#4`'s "optionally `executeOnApplication` / `inhibitionPolicy`" from optional to a MUST: `get_gas_info` is the only viable place to confirm those two CDO-default-prone fields, since the dump structurally cannot show a value equal to the default. `gas.set_effect_period`'s own return only echoes the input (which the HARD CONSTRAINTS warn is not proof of persistence). Recovery cost this task: ~9 calls + 1 destructive false-write cycle to verify a single default-valued boolean.
