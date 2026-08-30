---
id: E-gas-set-effect-tags-writes-deprecated-container-not-component
title: "gas.set_effect_tags writes only the deprecated InheritableOwnedTagsContainer, never a UTargetTagsGameplayEffectComponent — so a cooldown GE never grants tags (GetGrantedTags() stays empty) and the ability trips UE's cooldown-GE compile validation"
status: IN-REVIEW
severity: High
category: correctness
tags: [gas, set_effect_tags, granted-tags, gameplay-effect-components, cooldown, ue53-component-model, compile-validation, wiki]
---

# `gas.set_effect_tags` reports success but writes the wrong (deprecated) tag storage, so a cooldown GE never actually grants tags

`gas.set_effect_tags` (`GASHandler.cpp:2093-2099`) applies its resolved tags with
the **deprecated** legacy field only:

```cpp
for (const FGameplayTag& Tag : ResolvedTags)
{
    PRAGMA_DISABLE_DEPRECATION_WARNINGS
    EffectCDO->InheritableOwnedTagsContainer.AddTag(Tag);   // deprecated path
    PRAGMA_ENABLE_DEPRECATION_WARNINGS
}
```

It never adds a `UTargetTagsGameplayEffectComponent` to `EffectCDO->GEComponents`.
In UE 5.3+ the GameplayEffect tag model is **component-based**: the tags a GE
grants live in a `UTargetTagsGameplayEffectComponent` inside `GEComponents`; the
old `InheritableOwnedTagsContainer` is deprecated (hence the
`PRAGMA_DISABLE_DEPRECATION_WARNINGS` the handler itself wraps it in).

The consequence is a write that looks applied but is functionally inert for the
single most common use of granted tags — a **cooldown GameplayEffect**. UE's
cooldown-GE validation reads the **component** model (`CooldownGE->GetGrantedTags()`,
which returns `CachedGrantedTags` rebuilt from `GEComponents`, never from the
deprecated container), so a cooldown GE whose tags went only into the deprecated
container reads back empty and trips the validation at compile/save:

> `status Error: CooldownGameplayEffectClass 'GE Fireball Cooldown' grants no
> tags. A GameplayEffect class must grant tags (Component: Grant Tags to Target
> Actor) to be used as cooldown. (not saved)`

(This validation is a default-on warning gated behind the CVar
`AbilitySystem.WarnCooldownEffectWithoutTags`, and `UGameplayAbility::CheckCooldown`
is overridable — so it is the engine's default authoring guard, not a hard-coded
compile bar. In practice it blocks the canonical flow.)

In effect there is no MCP route to make a GE "grant tags via a component," so the
canonical "ability + cooldown GE" flow cannot be completed through the gateway at
all. It is also a **misleading-success + lying-readback**
pair: `set_effect_tags` echoes `tagsAdded:[Cooldown.Fireball]`, and
`gas.get_gas_info` then reports `grantedTags:[Cooldown.Fireball]` (it reads the
same deprecated `InheritableOwnedTagsContainer.CombinedTags` at
`GASHandler.cpp:2628-2635`), while the live CDO's `GEComponents` is `[]` — so
every signal the agent has says the tag is granted, but the compiler (and the
runtime) disagree.

## Distinct from the existing GAS tag ticket

`E-gas-set-effect-tags-drops-unregistered` (IN-REVIEW) is about **unregistered**
tags being silently dropped (now fixed to reject with `INVALID_PARAMS` +
`droppedTags`). That fix does not help here: in this task the tag
`Cooldown.Fireball` **was** registered first via `gameplay_tags.add` (call log
confirms it succeeded), the validate-before-mutate gate passed, and
`set_effect_tags` returned `tagsAdded:[Cooldown.Fireball]`. The defect is not
"tag rejected" — it is "registered tag written to the wrong storage class," so
the cooldown validation that reads `GEComponents` still fails. Different root
cause, different storage, different failure mode.

## What it should do / how to fix

- Make `gas.set_effect_tags` write the **component** model: find-or-add a
  `UTargetTagsGameplayEffectComponent` on `EffectCDO->GEComponents` and set its
  granted/target tags (the engine helper is
  `UTargetTagsGameplayEffectComponent::SetAndApplyTargetTagChanges` /
  the component's inheritable tags container), instead of (or in addition to,
  for back-compat) the deprecated `InheritableOwnedTagsContainer`. Then
  `gas.get_gas_info`'s `grantedTags` should read from the component so the
  readback matches the live CDO and the cooldown validation.
- Failing a full component-write, at minimum surface the prerequisite at the API
  level: a GE intended as a cooldown must grant tags via a component, and the
  current verb cannot author that — so the verb should not report unqualified
  success on a write the cooldown validator will reject.

### Docs gap (names the overlay page)

`docs/wiki-src/gas.md` has **no `set_effect_tags` section at all** and no mention
of the 5.3+ component-based granted-tags model or the cooldown-GE
"must grant tags via a component" requirement. Add a `set_effect_tags` section
that (a) states tags are authored into a `UTargetTagsGameplayEffectComponent`
(component model), and (b) calls out that a GameplayEffect used as an ability
cooldown must grant a tag or the ability's `blueprint.compile` will hard-fail
with "grants no tags." This is a downstream wiki edit, not this audit's job —
naming the page and the change is the deliverable.

## Evidence (this task — focus `gas.set_ability_cooldown`, Fireball ability-kit)

Friction note (verbatim): "gas.set_effect_tags echoes tagsAdded but writes
neither GEComponents (UTargetTagsGameplayEffectComponent) nor
InheritableOwnedTagsContainer, so the cooldown GE never grants tags and the
ability fails UE's cooldown-GE validation on compile (a blocker I
could not route around within constraints) ... Had to consult engine
GameplayEffect.h source to find ... the 5.3+ component-based granted-tags model."
(Engine note: the cooldown-GE check is `GetGrantedTags().IsEmpty()`, surfaced via
the default-on CVar `AbilitySystem.WarnCooldownEffectWithoutTags`; the deprecated
container is migrated into a component only by the one-time pre-Modular53
`ConvertTargetTagsComponent()` upgrade, which a freshly created 5.7 GE never
triggers — so the handler's deprecated write stays unmigrated and the readback
is empty.)

Call-log trace of the blocker (the tag was registered first, so this is not the
unregistered-drop case):

- `gameplay_tags.add {tag:"Cooldown.Fireball"}` -> ok (tag registered)
- `gas.set_effect_tags {GE_FireballCooldown grant Cooldown.Fireball}` ->
  ok, echoed `tagsAdded` — but "wrote nothing to CDO"
- `blueprint.compile {GE_FireballCooldown}` -> ok
- `blueprint.compile {GA_Fireball}` -> **Error: CooldownGameplayEffectClass
  'GE Fireball Cooldown' grants no tags ... (not saved)**
- `property.get {Default__GE_FireballCooldown_C, GEComponents}` -> `[]`
  (no TargetTags component)
- `property.get {Default__GE_FireballCooldown_C, InheritableOwnedTagsContainer}`
  -> "all containers empty"
- `gas.set_effect_tags {... Cooldown.Fireball}` (2nd attempt) -> echoed
  `tagsAdded` again, still no effect on `GEComponents`
- `gas.get_gas_info {GE_FireballCooldown}` -> `grantedTags:[Cooldown.Fireball]`
  while CDO `GEComponents:[]` ("stale" — readback disagrees with live CDO)

Cost: the ability could not be compiled/saved at all (task ended with a broken
hand-off), 2 wasted `set_effect_tags`, 2 failed `blueprint.compile`, ~4
`property.get` probes, plus out-of-band reading of engine `GameplayEffect.h` to
diagnose the component model — a hard blocker with no in-MCP workaround.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the `gas.set_ability_cooldown` Fireball ability-kit task (judge filed the separate cooldown/cost class-path silent-drop bug `B-set-ability-cooldown-cost-class-path-silent-drop`). Distinct PROCESS root cause not covered by any existing ticket: `gas.set_effect_tags` (`GASHandler.cpp:2093-2099`) writes tags ONLY into the deprecated `EffectCDO->InheritableOwnedTagsContainer` (wrapped in `PRAGMA_DISABLE_DEPRECATION_WARNINGS`), never adding a `UTargetTagsGameplayEffectComponent` to `GEComponents`, which is the UE 5.3+ component-based model that the mandatory cooldown-GE validator reads. Result: the registered tag `Cooldown.Fireball` was "accepted" (`tagsAdded` echoed; `get_gas_info` even reports `grantedTags:[Cooldown.Fireball]` reading the same deprecated container at `:2628-2635`), but the live CDO `GEComponents` stayed `[]`, so `blueprint.compile` of GA_Fireball hard-failed twice with "CooldownGameplayEffectClass ... grants no tags ... (not saved)" — a hard blocker with no MCP route to author the component, ending the task with a broken cooldown wiring. Explicitly distinct from `E-gas-set-effect-tags-drops-unregistered` (unregistered-tag silent-drop, now rejected by validate-before-mutate; here the tag WAS registered first and the gate passed). Proposed: write the component model (find-or-add `UTargetTagsGameplayEffectComponent`) so the GE actually grants tags and the readback matches the CDO; and add a `set_effect_tags` section to `docs/wiki-src/gas.md` (currently none) documenting the component model and the cooldown "must grant tags via a component" requirement.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded then fixed. REWORD: `category: ergonomic` → `correctness` (a write to the wrong storage class that the engine validator rejects is a correctness bug, not friction) and softened the "hard blocker / mandatory compile validation" framing to match the engine — the cooldown-GE check is `GetGrantedTags().IsEmpty()` surfaced via the default-on CVar `AbilitySystem.WarnCooldownEffectWithoutTags`, and `UGameplayAbility::CheckCooldown` is overridable, so it is the engine's default authoring guard (which does block the canonical flow), not a hard-coded compile bar. FIX (root cause): `gas.set_effect_tags` and the GE branch of `gas.add_tag_to_asset` now grant tags through the UE 5.3+ component model via a new shared helper `GrantTagsOnEffect()` — find-or-add `UTargetTagsGameplayEffectComponent` (header guarded by `__has_include`, falls back to the deprecated container on ≤5.2) and `SetAndApplyTargetTagChanges` (seeded additively from the component's configured tags), mirroring the engine's own `ConvertTargetTagsComponent()` bridge; it also keeps the deprecated container in sync for back-compat. `gas.get_gas_info` now reads `grantedTags` from `EffectCDO->GetGrantedTags()` (`CachedGrantedTags`, rebuilt from `GEComponents` — the exact source the cooldown validator reads) via a new `CollectGrantedTags()` helper, so the readback matches the live CDO. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Systems/GASHandler.cpp` (header include + two helpers + three call sites: set_effect_tags write, add_tag_to_asset GE branch, get_gas_info readback); docs `Docs/wiki-src/gas.md` (new "Granted tags & the cooldown-GE requirement" section documenting the component model and the cooldown requirement). Test: `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestGASHandlers.cpp` → `EditorAutomationRpcGateway.gas.set_effect_tags.GrantsViaComponent` — registers a tag transiently, calls `gas.set_effect_tags`, then asserts the live CDO `EffectCDO->GetGrantedTags()` is non-empty and `HasTagExact` the tag (the same `CachedGrantedTags` the cooldown validator reads), plus that `gas.get_gas_info`'s `grantedTags` reflects it. Counterfactual: revert to the deprecated-only `AddTag` and `GEComponents` stays empty → `GetGrantedTags()` empty → the assertion fails.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
