---
id: F-gas-ability-activation-tags-unauthorable
title: "No MCP route to author a GameplayAbility's ActivationBlockedTags / ActivationRequiredTags / ActivationOwnedTags — owner-state activation gating (\"can't cast while stunned/silenced/dead\") is unauthorable"
status: OPEN
severity: Medium
category: feature
tags: [gas, set_ability_tags, ability-tag-container-gap, activation-tags, gameplay-abilities]
encounters: 1
lastSeen: 2026-07-02T08:46:49.3078617+03:00
---

# GameplayAbility activation-gating tag containers are unauthorable via the MCP

`UGameplayAbility` carries several tag containers that gate *activation*:

- `ActivationBlockedTags` — the ability **cannot activate** while the owner has
  any of these tags (the canonical "can't cast while `State.Stunned` /
  `State.Silenced` / `State.Dead`" pattern).
- `ActivationRequiredTags` — the ability can activate **only** while the owner
  has all of these tags.
- `ActivationOwnedTags` — tags **granted to the owner** for the duration the
  ability is active.

None of these can be authored through any `gas.*` RPC. `gas.set_ability_tags`
(`GASHandler.cpp:1226-1232`) declares exactly three tag params, and they map to
three *different* containers:

```cpp
REGISTER_RPC_HANDLER("gas.set_ability_tags", "gas", "Set gameplay tags on a GameplayAbility",
    RPC_PARAMS(
        RPC_PARAM_REQ("blueprintPath", "string", "Path to the ability blueprint"),
        RPC_PARAM_OPT("abilityTags", "array", "Array of tag strings to add as ability tags"),           // -> AbilityTags (identity)
        RPC_PARAM_OPT("cancelAbilitiesWithTags", "array", "Array of tag strings for cancel tags"),       // -> CancelAbilitiesWithTag
        RPC_PARAM_OPT("blockAbilitiesWithTags", "array", "Array of tag strings for block tags")          // -> BlockAbilitiesWithTag
    ))
```

and the block param is applied to `BlockAbilitiesWithTag` (`GASHandler.cpp:1323-1326`):

```cpp
for (const FGameplayTag& Tag : ResolvedBlockTags)
{
    AddTagToAbilityContainer(AbilityCDO, FName(TEXT("BlockAbilitiesWithTag")), Tag);
}
```

The only other tag-writing verb, `gas.add_tag_to_asset`, writes a
GameplayAbility's identity container only (`GASHandler.cpp:2802-2804`:
`AbilityCDO->AbilityTags.AddTag(Tag)`). A whole-plugin grep for
`ActivationBlockedTags|ActivationRequiredTags|ActivationOwnedTags` under
`Plugins/PinWright/Source` returns **zero** hits — these fields are referenced
nowhere in the handler layer.

## Why the closest existing param is a wrong (not a) workaround

`BlockAbilitiesWithTag` and `ActivationBlockedTags` have **inverted** semantics:

- `BlockAbilitiesWithTag` = "while THIS ability is active, block OTHER abilities
  tagged X from activating."
- `ActivationBlockedTags` = "block THIS ability from activating while the owner
  already has tag X."

So "FlameBreath is blocked while the drake is stunned" belongs in
`ActivationBlockedTags` (owner has `State.Stunned` -> FlameBreath can't fire).
Routing `State.Stunned` into `blockAbilitiesWithTags` instead produces "while
FlameBreath is active, block abilities tagged `State.Stunned`" — which does
**not** prevent FlameBreath from firing when stunned. The requirement is
silently unmet, and a readback (`get_gas_info` / `asset.dump`) shows the tag
sitting in a "block" container, which can read as if the gating landed. There is
no correct in-MCP path, so the gap is worse than "blocker with a workaround."

## What it should do

Add optional params to `gas.set_ability_tags` (and/or a dedicated
`gas.set_ability_activation_tags`) covering the activation-gating containers:
`activationBlockedTags` -> `ActivationBlockedTags`, `activationRequiredTags` ->
`ActivationRequiredTags`, `activationOwnedTags` -> `ActivationOwnedTags` (these
last three live in `UGameplayAbility::ActivationBlockedTags` /
`ActivationRequiredTags` and the `ActivationOwnedTags` container — on 5.7 the
owned-tags are surfaced via the tag-container component but the ability-side
authoring fields still exist on the CDO). Validate-before-mutate against the
registry exactly like the existing three containers (reuse `ResolveTagsInto`).
Then `gas.get_gas_info` should read these back so the readback reflects them.

severity rationale: impact=capability-absent, only in-MCP path has inverted
semantics so there is no correct workaround (a state-gated ability cannot be
authored correctly at all) x reach=common ability-authoring pattern (activation
gating on owner state appears in most ability kits, but not every MCP session)
-> Medium.

## Verbatim repro (this task — Flame-Breath drake ability kit, seed gas.set_ability_targeting)

Task required: ability "blocked while the drake is stunned". Attempt call
(succeeded, but into the wrong container — the only one exposed):

- `gas.set_ability_tags {blueprintPath:/Game/GAS/Drake/GA_Drake_FlameBreath,
  abilityTags:[Ability.Drake.FlameBreath], blockAbilitiesWithTags:[State.Stunned]}`
  -> ok, `tagsAdded:[Ability.Drake.FlameBreath, State.Stunned]`

Attempt agent friction (verbatim): "gas.set_ability_tags only exposes
abilityTags/cancel/block, so 'blocked while stunned' had to go into
BlockAbilitiesWithTag (which in GAS means 'this ability blocks others with that
tag') rather than the semantically correct ActivationBlockedTags, which the tool
does not surface."

Confirmed by source: `gas.set_ability_tags` param schema (`GASHandler.cpp:1226-1232`),
its block-tag application (`:1323-1326`), and `gas.add_tag_to_asset`'s
ability branch (`:2802-2804`) collectively expose only AbilityTags /
CancelAbilitiesWithTag / BlockAbilitiesWithTag; no `gas.*` handler writes any
Activation* container.

## History
- `#1-initial-repro` `OPEN` reporter — Filed from the Flame-Breath drake ability-kit task (seed `gas.set_ability_targeting`; culprit is the neighbor `gas.set_ability_tags`). A reasonable, common GAS requirement — "ability blocked while owner is stunned" (`ActivationBlockedTags`) — is unauthorable: `gas.set_ability_tags` exposes only AbilityTags/CancelAbilitiesWithTag/BlockAbilitiesWithTag, `gas.add_tag_to_asset` writes only the identity container, and `ActivationBlockedTags`/`ActivationRequiredTags`/`ActivationOwnedTags` appear nowhere in the plugin Source. The nearest param (`blockAbilitiesWithTags` -> `BlockAbilitiesWithTag`) has inverted semantics, so the agent's forced workaround produces a silently-wrong ability that does NOT gate on stun. Proposed: add `activationBlockedTags`/`activationRequiredTags`/`activationOwnedTags` params (validate-before-mutate via `ResolveTagsInto`) and surface them in `gas.get_gas_info` readback.
