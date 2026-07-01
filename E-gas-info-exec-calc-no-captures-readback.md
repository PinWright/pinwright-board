---
id: E-gas-info-exec-calc-no-captures-readback
title: "gas.get_gas_info returns bare metadata for a GameplayEffectExecutionCalculation blueprint (no gasType, no RelevantAttributesToCapture) — capture readback is unsatisfiable"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [gas, get_gas_info, execution-calculation, set_execution_capture, captures, readback, inspect-after-mutate]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# `gas.get_gas_info` can't read back the captures on a GameplayEffectExecutionCalculation

This is the **GameplayEffectExecutionCalculation (exec-calc) sibling** of the
`gas.get_gas_info` readback-thinness family. The three existing tickets each widen
a *different* `gasType` branch — `E-gas-info-skips-asc-owner-actor` (ASC-owner +
the GameplayEffect branch's executions/tags), `E-gas-info-ability-omits-cooldown-cost-tags`
(GameplayAbility branch), `E-gas-info-effect-modifiers-duration-value-thin`
(GameplayEffect per-modifier/duration + AttributeSet attributes). **None of them
touches the exec-calc asset itself**, which has no `gasType` branch at all.

For a `UGameplayEffectExecutionCalculation` blueprint — the asset that
`gas.create_execution_calculation` creates and `gas.set_execution_capture`
populates — `get_gas_info` falls all the way through the `else if` chain (no
`UGameplayEffectExecutionCalculation` / `UGameplayEffectCalculation` branch
exists) into `CollectAbilitySystemOwnerInfo`, which finds no ASC on an exec calc,
so the response is bare blueprint metadata: `assetPath`, `assetName`,
`class:"Blueprint"`, `type:"Blueprint"`, `generatedClass`, `parentClass` — **no
`gasType`, and no `RelevantAttributesToCapture` / captures field**.

So an exec-calc authoring task whose named final step is "read back the GAS info
on the execution calc so I can confirm the captures (AttackPower from Source
snapshot=true, Armor from Target snapshot=false) are in place" **cannot be
satisfied** by `get_gas_info`: it returns a clean success with no `gasType` and
nothing about the captures. The captures provably persist (the
`gas.set_execution_capture` write succeeds) but the only way to read them back is
an out-of-band `property.get` on `Default__<Exec>_C.RelevantAttributesToCapture`
— `gas.set_execution_capture` itself echoes only `captureCount`, not the
per-capture attribute/source/snapshot, so even the write response can't confirm
*which* attribute landed on *which* source.

This is the same false-negative shape the family already recognizes: a tool
named/described as the GAS-info readback returns a clean success that looks as
though the asset has nothing GAS-related, when it is a fully-configured execution
calc with captures.

## Asymmetry that makes it concrete

After wiring the exec calc into a GE, `get_gas_info` on the **GE** confirms the
attachment (the `E-gas-info-skips-asc-owner-actor` #2 fix landed
`executionClasses`), but `get_gas_info` on the **exec calc** confirms nothing —
so the inputs to the calc (the captures) are invisible exactly where you'd look
for them.

## Verbatim repro (live, this iteration)

Assets under `/Game/GAS` (DamageMitigationExec authored with two captures via
`gas.set_execution_capture`: AttackPower Source/snapshot=true, Armor
Target/snapshot=false; wired into GE_ApplyDamage):

- `gas.get_gas_info {assetPath:"/Game/GAS/DamageMitigationExec"}` ->
  `{"assetPath":"/Game/GAS/DamageMitigationExec","assetName":"DamageMitigationExec","class":"Blueprint","type":"Blueprint","generatedClass":"DamageMitigationExec_C","parentClass":"GameplayEffectExecutionCalculation"}`
  (no `gasType`, no captures / `RelevantAttributesToCapture`)

The GE that the exec calc is wired into, by contrast, *does* confirm the
attachment side:

- `gas.get_gas_info {assetPath:"/Game/GAS/GE_ApplyDamage"}` ->
  `{...,"gasType":"GameplayEffect","durationPolicyName":"Instant",...,"executionCount":1,"executionClasses":["DamageMitigationExec_C"],"grantedTags":[]}`

(Passing the generated-class path `/Game/GAS/DamageMitigationExec_C` to
`get_gas_info` errors `[NOT_FOUND] Asset not found` — it takes the asset path, not
the class path — so there is no exec-calc readback at all via this tool.)

## Root cause

`gas.get_gas_info` (`GASHandler.cpp`, blueprint branch ~`:2872-3010`) sets
`gasType` only for `UGameplayAbility` / `UGameplayEffect` / `UAttributeSet` /
`UGameplayCueNotify_Static` / `AGameplayCueNotify_Actor`, then falls through to
`CollectAbilitySystemOwnerInfo`. There is no
`ParentClass->IsChildOf(UGameplayEffectExecutionCalculation::StaticClass())`
branch, so an exec-calc blueprint never gets a `gasType` and its
`RelevantAttributesToCapture` is never read. The captures live on the CDO as an
`FArrayProperty` named `RelevantAttributesToCapture` (the same property
`gas.set_execution_capture` writes via reflection at `GASHandler.cpp:3776`), each
element an `FGameplayEffectAttributeCaptureDefinition`
(AttributeToCapture / AttributeSource / bSnapshot).

**Workaround:** confirm the captures with `property.get` on
`Default__<Exec>_C.RelevantAttributesToCapture` instead of `gas.get_gas_info`.

**Fix (proposed):** add a branch to `gas.get_gas_info` for
`UGameplayEffectExecutionCalculation` (or its base `UGameplayEffectCalculation`)
blueprints — stamp `gasType:"GameplayEffectExecutionCalculation"` and emit a
`captures` array, one entry per `RelevantAttributesToCapture` element with
`attribute` (the `FGameplayAttribute` name), `source` ("source"/"target" from
`AttributeSource`), and `snapshot` (bool), read back via the same reflection
pattern `gas.set_execution_capture` writes through. Mirror the `## Inspect-after-mutate`
note in `docs/wiki-src/gas.md` (currently it documents GameplayEffect /
AttributeSet / GameplayAbility / ASC-owner gasTypes but not the exec calc).
Secondary: `gas.set_execution_capture` could echo the resolved per-capture
attribute/source/snapshot (not just `captureCount`) so the write response also
confirms which attribute landed where.

## History
- `#2-exec-calc-captures-readback` `IN-REVIEW` developer — Added a `UGameplayEffectExecutionCalculation` branch to `gas.get_gas_info`'s blueprint else-if chain (`GASHandler.cpp`, before the final `CollectAbilitySystemOwnerInfo` fall-through): it now stamps `gasType:"GameplayEffectExecutionCalculation"` and emits a `captures` array via a new static helper `CollectExecutionCaptures(GeneratedClass)`. The helper reads `RelevantAttributesToCapture` off the exec-calc CDO through the SAME `FArrayProperty`/`FScriptArrayHelper` reflection path `gas.set_execution_capture` writes through (the field is protected on `UGameplayEffectCalculation`), reinterpreting each element as `FGameplayEffectAttributeCaptureDefinition` and emitting `{attribute, source ("source"/"target"), snapshot}`. Branched on `UGameplayEffectExecutionCalculation` specifically (not the `UGameplayEffectCalculation` base) so the gasType label stays accurate and MMC blueprints aren't mislabelled. Updated `docs/wiki-src/gas.md` inspect-after-mutate section with the exec-calc gasType bullet. Files: `Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`, `Docs/wiki-src/gas.md`. Test: `PinWright.gas.get_gas_info.ExecCalcEmitsCaptures` in `Source/PinWright/Private/Tests/Gameplay/TestGASHandlers.cpp` — creates an AttributeSet (AttackPower + Armor), an exec calc, writes two captures (AttackPower Source snapshot=true, Armor Target snapshot=false) via `gas.set_execution_capture`, then asserts `get_gas_info` returns `gasType:"GameplayEffectExecutionCalculation"` and a two-entry `captures` array with matching attribute/source/snapshot; reverting the branch drops gasType + captures and fails it.
- `#1-initial-repro` `OPEN` reporter — Seed-mode struggle (focus `gas.create_execution_calculation`; culprit `gas.get_gas_info`). A damage-mitigation exec-calc task's named step 7 — "read back the GAS info on DamageMitigationExec to confirm the captures are in place" — is unsatisfiable: `gas.get_gas_info` on the exec calc returns bare blueprint metadata (`generatedClass:"DamageMitigationExec_C"`, `parentClass:"GameplayEffectExecutionCalculation"`) with no `gasType` and no `RelevantAttributesToCapture` field, because the handler's blueprint branch has no exec-calc `else if` and falls through to the ASC-owner path. Replayed live this iteration: verbatim responses above. The captures provably persist (`gas.set_execution_capture` succeeded with `captureCount:2`) and the GE-side wiring IS confirmable (`get_gas_info {GE_ApplyDamage}` -> `executionClasses:["DamageMitigationExec_C"]`), but the captures on the exec calc itself are only readable via `property.get` on the CDO. Distinct from the three existing `get_gas_info` tickets (ASC-owner/GE-executions, GameplayAbility, GE-modifiers/AttributeSet) — none adds an exec-calc branch. Proposed: add a `gasType:"GameplayEffectExecutionCalculation"` branch emitting a `captures` array (attribute/source/snapshot) read from `RelevantAttributesToCapture`.
