---
id: F-gas-ability-activation-tags-unauthorable
title: "gas.set_ability_tags has no activation-gating tag params (ActivationBlockedTags / ActivationRequiredTags / ActivationOwnedTags) — authorable today only via a source-dive blueprint.set_default; the nearest gas param has inverted semantics and get_gas_info can't read them back"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [gas, set_ability_tags, get_gas_info, ability-tag-container-gap, activation-tags, gameplay-abilities, discoverability, validation, readback]
encounters: 1
lastSeen: 2026-07-02T08:46:49.3078617+03:00
---

# `gas.set_ability_tags` exposes no activation-gating tag params

`UGameplayAbility` carries three tag containers that gate *activation* (all plain,
non-deprecated `FGameplayTagContainer` `UPROPERTY`s on the ability —
`GameplayAbility.h:766/770/774`):

- `ActivationBlockedTags` — the ability **cannot activate** while the owner has
  any of these tags (the canonical "can't cast while `State.Stunned` /
  `State.Silenced` / `State.Dead`" pattern).
- `ActivationRequiredTags` — the ability can activate **only** while the owner
  has all of these tags.
- `ActivationOwnedTags` — tags **granted to the owner** for the duration the
  ability is active.

None of these has a dedicated `gas.*` param. `gas.set_ability_tags`
(`GASHandler.cpp:1226-1232`) declares exactly three tag params that map to three
*different* containers — `abilityTags` -> identity `AbilityTags`,
`cancelAbilitiesWithTags` -> `CancelAbilitiesWithTag`, `blockAbilitiesWithTags`
-> `BlockAbilitiesWithTag` (`:1319-1326`) — and the only other tag-writing verb,
`gas.add_tag_to_asset`, writes the identity container only
(`:2802-2804`, `AbilityCDO->AbilityTags.AddTag(Tag)`). A whole-plugin grep for
`ActivationBlockedTags|ActivationRequiredTags|ActivationOwnedTags` under
`Plugins/PinWright/Source` returns **zero** hits — no `gas.*` handler names them.

## This is NOT "unauthorable" — a correct in-MCP path already exists

(The original filing claimed these containers are *unauthorable* and that "there
is no correct in-MCP path." That is **false** and is corrected here.) The generic
reflection setter `blueprint.set_default` (`BlueprintPropertyHandler.cpp:390-491`)
authors them correctly: it grabs the ability CDO
(`Blueprint->GeneratedClass->GetDefaultObject()`, `:436`), resolves **any**
UPROPERTY by name against the full class hierarchy including inherited native
fields (`CDO->GetClass()->FindPropertyByName(*PropertyName)`, `:454`), and writes
via `ApplyJsonValueToProperty` (`:471`), whose `FStructProperty` branch imports an
`FGameplayTagContainer` from a JSON-object or an ExportText literal
(`PropertyImport.cpp:784-877`, `ImportTextToProperty` fallback `:865`). So

```
blueprint.set_default {path:/Game/GAS/Drake/GA_Drake_FlameBreath,
  propertyName:"ActivationBlockedTags",
  value:"(GameplayTags=((TagName=\"State.Stunned\")))"}
```

writes the **semantically correct native container** — not the inverted
`BlockAbilitiesWithTag`. The board already treats this as the standard escape
hatch: `E-ai-assign-verbs-need-set-default-fallback.md` documents the identical
`blueprint.set_default {path, propertyName, value}` pattern for a native CDO
property a dedicated verb doesn't cover.

## The real, residual gap (why this is still worth fixing)

The capability exists, but the ergonomics are poor and one path is an active trap:

1. **Inverted-semantics trap (the sharp edge).** `BlockAbilitiesWithTag` and
   `ActivationBlockedTags` are inverted: `blockAbilitiesWithTags` means "while
   THIS ability is active, block OTHER abilities tagged X," whereas the requirement
   ("FlameBreath is blocked while the drake is stunned") needs `ActivationBlockedTags`
   ("block THIS ability while the owner already has `State.Stunned`"). An agent
   reaching for the obvious `gas.set_ability_tags` verb routes `State.Stunned`
   into `blockAbilitiesWithTags`, the call succeeds
   (`tagsAdded:[...State.Stunned]`), and the ability is **silently not gated** on
   stun — a readback shows the tag sitting in a "block" container, reading as if
   the gating landed.
2. **Discoverability / source-dive.** The correct path
   (`blueprint.set_default` with the exact native field name +
   `FGameplayTagContainer` ExportText literal) is not derivable from the `gas.*`
   surface; it requires a source dive into `GameplayAbility.h`.
3. **No registry validation on the generic path.** `blueprint.set_default` /
   ImportText do not registry-validate tags the way `gas.set_ability_tags` does
   via `ResolveTagsInto` (`GASHandler.cpp:1275/1280/1285` -> reject with
   `droppedTags`); an unregistered tag imports as an **empty** container rather
   than being rejected.
4. **No readback.** `gas.get_gas_info`'s GameplayAbility branch
   (`GASHandler.cpp:2940-2977`) surfaces instancing/net/cooldown/cost/`abilityTags`
   but not these three containers, so confirming the gating landed forces an
   `asset.dump` / `property.get` fallback.

## What it should do (fix)

Add three optional params to `gas.set_ability_tags` — `activationOwnedTags` ->
`ActivationOwnedTags`, `activationRequiredTags` -> `ActivationRequiredTags`,
`activationBlockedTags` -> `ActivationBlockedTags` — validated-before-mutate via
the existing shared `ResolveTagsInto` (so an unregistered tag is rejected with
`INVALID_PARAMS` + `droppedTags`, exactly like the current three containers) and
applied through the existing generic reflection helper
`AddTagToAbilityContainer` (`GASHandler.cpp:466-480`, `FindPropertyByName` +
`FGameplayTagContainer` type-check). Then surface the three containers in
`gas.get_gas_info`'s GameplayAbility branch so the readback reflects them and the
inverted-`BlockAbilitiesWithTag` trap becomes unnecessary. This is dedicated,
validated, discoverable, readable-back sugar over the already-working
`blueprint.set_default` path (distinct from the read-side sibling
`E-gas-info-ability-omits-cooldown-cost-tags`, which widens cooldown/cost/tags/
targeting readback and does **not** touch the activation containers).

severity rationale: impact=**soft blocker** — a correct path exists but only via
a source-dive `blueprint.set_default` ExportText literal (the obvious `gas.*` verb
has an inverted-semantics silent-wrong trap), and `get_gas_info` omits the field
so confirming the gating forces a fallback (per the board rubric this is squarely
the "documented workaround / source dive / readback omits a field" Medium class,
NOT the capability-absent "no workaround" class the original filing claimed) x
reach=common ability-authoring pattern (activation gating on owner state appears
in most ability kits, but not every MCP session) -> Medium.

## Verbatim repro (this task — Flame-Breath drake ability kit, seed gas.set_ability_targeting)

Task required: ability "blocked while the drake is stunned". Attempt call
(succeeded, but into the wrong container — the only `gas.*` one exposed):

- `gas.set_ability_tags {blueprintPath:/Game/GAS/Drake/GA_Drake_FlameBreath,
  abilityTags:[Ability.Drake.FlameBreath], blockAbilitiesWithTags:[State.Stunned]}`
  -> ok, `tagsAdded:[Ability.Drake.FlameBreath, State.Stunned]`

Attempt agent friction (verbatim): "gas.set_ability_tags only exposes
abilityTags/cancel/block, so 'blocked while stunned' had to go into
BlockAbilitiesWithTag (which in GAS means 'this ability blocks others with that
tag') rather than the semantically correct ActivationBlockedTags, which the tool
does not surface." (The agent did not reach for `blueprint.set_default`, the
correct existing path — a discoverability failure, not a capability-absent one.)

## History
- `#1-initial-repro` `OPEN` reporter — Filed from the Flame-Breath drake ability-kit task (seed `gas.set_ability_targeting`; culprit is the neighbor `gas.set_ability_tags`). A reasonable, common GAS requirement — "ability blocked while owner is stunned" (`ActivationBlockedTags`) — is unauthorable: `gas.set_ability_tags` exposes only AbilityTags/CancelAbilitiesWithTag/BlockAbilitiesWithTag, `gas.add_tag_to_asset` writes only the identity container, and `ActivationBlockedTags`/`ActivationRequiredTags`/`ActivationOwnedTags` appear nowhere in the plugin Source. The nearest param (`blockAbilitiesWithTags` -> `BlockAbilitiesWithTag`) has inverted semantics, so the agent's forced workaround produces a silently-wrong ability that does NOT gate on stun. Proposed: add `activationBlockedTags`/`activationRequiredTags`/`activationOwnedTags` params (validate-before-mutate via `ResolveTagsInto`) and surface them in `gas.get_gas_info` readback.
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORD + implement. Corrected the false central premise: these containers are NOT "unauthorable" — `blueprint.set_default {path, propertyName:"ActivationBlockedTags", value:"(GameplayTags=(...))"}` already authors the correct native `FGameplayTagContainer` CDO fields by generic reflection (`BlueprintPropertyHandler.cpp:454`+`471`, struct ImportText fallback `PropertyImport.cpp:865`; the exact `set_default` escape hatch documented in `E-ai-assign-verbs-need-set-default-fallback`). Reframed from "unauthorable / no correct path" (capability-absent) to a soft-blocker ergonomics gap: inverted-semantics trap in the nearest `gas.*` param + source-dive discoverability + no registry validation on the generic path + no `get_gas_info` readback; severity kept Medium but re-justified on the rubric's "source dive / readback omits a field" grounds; category bug/feature -> ergonomic. Implemented the dedicated sugar: added `activationOwnedTags`/`activationRequiredTags`/`activationBlockedTags` params to `gas.set_ability_tags` (validated via the shared `ResolveTagsInto` -> `droppedTags` rejection, written via the existing generic `AddTagToAbilityContainer` reflection helper), and surfaced all three containers in `gas.get_gas_info`'s GameplayAbility branch via a new symmetric reflection reader `CollectAbilityContainerTags`. File: `Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`. Regression test `PinWright.gas.set_ability_tags.ActivationTagsAuthoredAndReadBack` (`Source/PinWright/Private/Tests/Gameplay/TestGASHandlers.cpp`): creates an ability, registers three transient tags, routes one into each activation param via `gas.set_ability_tags`, then asserts `gas.get_gas_info` echoes each tag under `activationOwnedTags`/`activationRequiredTags`/`activationBlockedTags` — fails if either the write params or the readback emits are reverted (empty container -> empty array -> membership assertion misses). Distinct from the read-side sibling `E-gas-info-ability-omits-cooldown-cost-tags` (cooldown/cost/tags/targeting readback; does not touch the activation containers). Did not compile/run the full suite (later phase).
