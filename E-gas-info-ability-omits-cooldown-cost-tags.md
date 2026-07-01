---
id: E-gas-info-ability-omits-cooldown-cost-tags
title: "gas.get_gas_info GameplayAbility branch emits only instancing/netExecution policy — never cooldownEffect / costEffect / abilityTags, so the canonical 'confirm the cooldown and cost are wired' readback is unsatisfiable"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, gas, get_gas_info, gameplay-ability, cooldown, cost, ability-tags, readback, inspect-after-mutate, wiki]
---

# `gas.get_gas_info`'s GameplayAbility branch can't confirm the ability's cooldown / cost / tags

This is the **GameplayAbility sibling** of the GameplayEffect/AttributeSet/ASC-owner
readback-thinness already tracked under `E-gas-info-skips-asc-owner-actor`
(IN-REVIEW) and `E-gas-info-effect-modifiers-duration-value-thin` (OPEN): the same
"documented readback omits exactly the field the task wants to confirm" shape, here
on the **GameplayAbility** `gasType` branch, which none of those tickets touch.

For a GameplayAbility blueprint the `get_gas_info` ability branch
(`GASHandler.cpp:2691-2720`) emits only `gasType:"GameplayAbility"`,
`instancingPolicy`, and `netExecutionPolicy`. It never reads from the ability CDO:

- **`CooldownGameplayEffectClass`** — the cooldown GE wired by
  `gas.set_ability_cooldown`.
- **`CostGameplayEffectClass`** — the cost GE wired by `gas.set_ability_costs`.
- **`AbilityTags`** — the tags set by `gas.set_ability_tags`.

So an ability-authoring task whose named final step is "read back the GAS info for
GA_Fireball so I can confirm the cooldown and cost effects are correctly attached"
**cannot be satisfied** by `get_gas_info`: it returns a clean success with none of
the three fields the success-check asks for. The agent must pivot to out-of-band
`property.get` on `Default__<Ability>_C.CooldownGameplayEffectClass` /
`CostGameplayEffectClass` to prove the wiring — which is also the only way to catch
the `set_ability_cooldown`/`set_ability_costs` silent-drop
(`B-set-ability-cooldown-cost-class-path-silent-drop`); the two gaps compound:
the readback can't see the wiring **and** the write can silently no-op, so a
broken ability looks fully configured.

## Distinct from existing GAS readback tickets

- `E-gas-info-skips-asc-owner-actor` (IN-REVIEW) — added an `AbilitySystemOwner`
  branch and widened the **GameplayEffect** branch (executions/tags). Its scope is
  the ASC-owner actor and the GameplayEffect branch; it does **not** touch the
  GameplayAbility branch's cooldown/cost/tags.
- `E-gas-info-effect-modifiers-duration-value-thin` (OPEN) — per-modifier
  op/magnitude + duration value + AttributeSet attributes, all on the
  **GameplayEffect / AttributeSet** branches. Not the ability branch.
- `B-set-ability-cooldown-cost-class-path-silent-drop` (judge-filed, IN-REVIEW) —
  the **write-side** silent no-op on the cooldown/cost setters. This is the
  **read-side** omission that hides whether that write took.

## What it should do / how to fix

Source widening (right where the branch already stamps instancing/netExecution at
`GASHandler.cpp:2691-2720`): read the ability CDO's `CooldownGameplayEffectClass`
and `CostGameplayEffectClass` (`TSubclassOf<UGameplayEffect>`) and emit their
class paths (or null), plus an `abilityTags` array from `AbilityCDO->AbilityTags`
(the 5.7 deprecated-container read already mirrored by `gas.set_ability_tags` at
`:1060-1082`, via `GetAssetTags()`). Then `get_gas_info` on an ability confirms
the cooldown/cost wiring and the tags in one call.

### Docs gap (names the overlay page)

`docs/wiki-src/gas.md` has **no `get_gas_info` / `## Inspect-after-mutate` section
at all** (the same gap `E-gas-info-effect-modifiers-duration-value-thin` notes for
the GE branch). Until the source is widened, that section should state that the
GameplayAbility branch returns only `gasType` / `instancingPolicy` /
`netExecutionPolicy` and that confirming an ability's cooldown/cost wiring today
requires `property.get` on `Default__<Ability>_C.CooldownGameplayEffectClass` /
`CostGameplayEffectClass`. Downstream wiki edit, not this audit's job — naming the
page and change is the deliverable.

## Evidence (this task — focus `gas.set_ability_cooldown`, Fireball ability-kit)

Friction note (verbatim): "gas.get_gas_info on an ability omits cooldown/cost/tags
entirely (can't confirm the named success check)."

Call-log trace: after wiring, `gas.get_gas_info {GA_Fireball}` returned with "no
cooldown/cost fields in output" / "no cooldown/cost/tags fields reported," so the
agent fell back to `property.get` on `Default__GA_Fireball_C` for both
`CooldownGameplayEffectClass` and `CostGameplayEffectClass` (the readbacks that
both exposed the class-path silent-drop and, later, confirmed the `.._C`-path
round-trip). The story's explicit step 7 ("read back the GAS info ... to confirm
the cooldown and cost effects are correctly attached") was not satisfiable via the
named tool.

Cost: the named success-check tool returned nothing usable; the agent emitted ~4
`property.get` calls (2 that errored on the bare `_C` UClass before targeting the
`Default__..._C` CDO, 2 that succeeded) to substitute for the readback.

## History
- `#3-already-resolved` `IN-REVIEW` developer — Re-analysis (lead): all three validity lenses voted valid and the fix is already present and committed in current source (commit `8cf262a`), so no code change this pass. Verified in source: the two reflection helpers `CollectAbilityTags` (`GASHandler.cpp:245-267`, `GetAssetTags()` on 5.7+ / deprecated `AbilityTags` container pre-5.7) and `ReadAbilityEffectClassPath` (`GASHandler.cpp:274-282`, `TSubclassOf<UGameplayEffect>` → object path or ""); the GameplayAbility branch emits `cooldownEffect` / `costEffect` / `abilityTags` at `GASHandler.cpp:2770-2775` right after the instancing/netExecution policy stamps; and the regression test `PinWright.gas.get_gas_info.AbilityEmitsCooldownCostTags` at `TestGASHandlers.cpp:851`. Status left at IN-REVIEW for a tester to verify-green; lease released. (Note: file path is `Source/PinWright/Private/Handlers/Systems/GASHandler.cpp` — the `Systems/` subdir — which the `#2` entry's prose elided.)
- `#2-readback-widening` `IN-REVIEW` developer — REWORD+implement. Corrected stale citations in the body (the get_gas_info ability branch is `GASHandler.cpp:2691-2720`, not `:2560-2589` which is `add_tag_to_asset`; `set_ability_tags` is `:1060-1082`, not `:929-948`; the write-side B-ticket is IN-REVIEW, not OPEN). Source fix in `Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`: added two reflection helpers next to `CollectGrantedTags`/`GetAbilityPropertyValue` — `CollectAbilityTags` (reads `AbilityTags` via `GetAssetTags()` on 5.7+, deprecated container pre-5.7) and `ReadAbilityEffectClassPath` (reads a `TSubclassOf<UGameplayEffect>` UPROPERTY → object path or "") — then widened the GameplayAbility branch (right after it stamps instancing/netExecution policy) to emit `cooldownEffect` / `costEffect` (the CDO's `CooldownGameplayEffectClass` / `CostGameplayEffectClass` object paths) and an `abilityTags` array. Mirrors the already-landed GE-branch `executionClasses`/`grantedTags` widening and reuses the same reflection pattern the setters write through. Regression test `PinWright.gas.get_gas_info.AbilityEmitsCooldownCostTags` in `Source/PinWright/Private/Tests/Gameplay/TestGASHandlers.cpp`: creates an ability + cooldown/cost GEs, wires them via `set_ability_cooldown`/`set_ability_costs`, sets a transient `abilityTags` tag, then asserts `get_gas_info` echoes `cooldownEffect`/`costEffect` (matching the persisted `_C` class paths) and `abilityTags` — reverting the three field emits fails it. Docs (`docs/wiki-src/gas.md` `## Inspect-after-mutate` section) remain a downstream wiki edit, not this audit's job.
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the `gas.set_ability_cooldown` Fireball ability-kit task (judge filed the write-side `B-set-ability-cooldown-cost-class-path-silent-drop`). Distinct read-side PROCESS gap: `get_gas_info`'s GameplayAbility branch (`GASHandler.cpp:2560-2589`) emits only `gasType`/`instancingPolicy`/`netExecutionPolicy` and never reads the CDO's `CooldownGameplayEffectClass`, `CostGameplayEffectClass`, or `AbilityTags`, so the story's named step-7 readback ("confirm the cooldown and cost effects are correctly attached") is unsatisfiable via the tool — the agent pivoted to `property.get` on `Default__GA_Fireball_C` (the same probe that exposed the class-path silent-drop). Distinct from `E-gas-info-skips-asc-owner-actor` (ASC-owner + GE branch) and `E-gas-info-effect-modifiers-duration-value-thin` (GE/AttributeSet branches) — neither touches the ability branch — and from the write-side `B-set-ability-cooldown-cost-class-path-silent-drop`. Proposed: widen the ability branch to emit `cooldownEffect`/`costEffect` class paths + an `abilityTags` array; and add a `get_gas_info`/`## Inspect-after-mutate` section to `docs/wiki-src/gas.md` (currently none) documenting the ability-branch fields and the `property.get` fallback.
