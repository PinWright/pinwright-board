---
id: E-gas-info-skips-asc-owner-actor
title: "gas.get_gas_info returns bare blueprint metadata for ASC-owning actors (no gasType / ASC component / replicationMode)"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [gas, get_gas_info, asc, readback]
---

# `gas.get_gas_info` reports no GAS data for an actor blueprint that owns an ASC

`gas.get_gas_info` is documented as "Get GAS information about an asset." For
the four GAS *asset* parent classes it does this well: it stamps a `gasType`
and the relevant detail fields. But for an **actor/Character blueprint that
owns an AbilitySystemComponent** (the most common real GAS subject — the pawn
that the ASC, attribute set, and granted abilities are all *for*), it returns
only bare blueprint metadata: `assetPath`, `assetName`, `class`, `type`,
`generatedClass`, `parentClass` — **no `gasType`, no ASC component, no
replicationMode, no granted abilities**, and no signal that it skipped the
actor's GAS configuration.

This is concretely misleading: a tool named/described as the GAS-info readback
returns a clean success that looks as though the asset has nothing GAS-related,
when in fact the asset is fully GAS-configured. A caller using
`get_gas_info` as a success-check for `gas.add_ability_system_component` /
`gas.configure_asc` / `gas.grant_ability` gets a false-negative readback and
must fall back to `blueprint.scs.get` + `blueprint.get` to prove the ASC and
the grant actually persisted.

The data is genuinely present — only the readback omits it. Verified live:
`blueprint.scs.get` on the same blueprint independently shows the ASC component.

## Verbatim repro (live, this iteration)

Scaffold built earlier under `/Game/Heroes`: `BP_GASHero` (parent `Character`)
with an `AbilitySystem` ASC (`configure_asc` replicationMode=mixed), a granted
`GA_Dash` ability, plus sibling GAS assets `AS_HeroAttributes` and `GA_Dash`.

The GAS *asset* siblings get a `gasType` and detail fields:

- `gas.get_gas_info {assetPath:"/Game/Heroes/AS_HeroAttributes"}` ->
  `{"assetPath":"/Game/Heroes/AS_HeroAttributes","assetName":"AS_HeroAttributes","class":"Blueprint","type":"Blueprint","generatedClass":"AS_HeroAttributes_C","parentClass":"AttributeSet","gasType":"AttributeSet"}`
- `gas.get_gas_info {assetPath:"/Game/Heroes/Abilities/GA_Dash"}` ->
  `{"assetPath":"/Game/Heroes/Abilities/GA_Dash","assetName":"GA_Dash","class":"Blueprint","type":"Blueprint","generatedClass":"GA_Dash_C","parentClass":"GameplayAbility","gasType":"GameplayAbility","instancingPolicy":2,"netExecutionPolicy":0}`

But the ASC-owning Character — the asset that actually has the ASC, the
replication mode, and the granted ability — gets nothing GAS:

- `gas.get_gas_info {assetPath:"/Game/Heroes/BP_GASHero"}` ->
  `{"assetPath":"/Game/Heroes/BP_GASHero","assetName":"BP_GASHero","class":"Blueprint","type":"Blueprint","generatedClass":"BP_GASHero_C","parentClass":"Character"}`

  (no `gasType`, no ASC, no `replicationMode`, no abilities)

Independent proof the data persisted (so this is purely a readback omission,
not missing data):

- `blueprint.scs.get {blueprintPath:"/Game/Heroes/BP_GASHero"}` ->
  components include `{"name":"AbilitySystem","class":"AbilitySystemComponent","source":"scs",...}`

## Root cause

`GASHandler.cpp` `gas.get_gas_info` (handler at ~line 2236) only sets `gasType`
when `ParentClass->IsChildOf` one of `UGameplayAbility` / `UGameplayEffect` /
`UAttributeSet` / `UGameplayCueNotify_Static` / `AGameplayCueNotify_Actor`.
There is no branch that detects an actor blueprint owning an
`UAbilitySystemComponent` (via SCS or native), so such blueprints fall through
to the bare-metadata path with no `gasType`.

**Workaround:** cross-check ASC presence with `blueprint.scs.get` and granted
abilities with `blueprint.get` instead of relying on `get_gas_info`.

**Fix (proposed):** add a branch for ASC-owning actor blueprints — scan the
generated class' SCS + native components for a `UAbilitySystemComponent` (reuse
the SCS-plus-CDO detection already in this file at the `grant_ability` handler),
and when found emit `gasType:"AbilitySystemOwner"` plus an
`abilitySystemComponents` list (component name + replication mode) so the
readback no longer silently reports an ASC-owning actor as having no GAS data.

Scope note: do **not** add a "granted abilities" field here. `gas.grant_ability`
only creates an empty `InitialAbilities` array variable on the actor blueprint
and never records a per-ability grant on the CDO (it instructs the caller to
call `GiveAbility` at runtime), so there is no structurally-present per-ability
grant for `get_gas_info` to read back — a granted-abilities field would honestly
only ever echo an empty list. Granted-ability readback is out of scope for this
ticket.

## History
- `#1-initial-repro` `OPEN` reporter — get_gas_info emits gasType + detail for AttributeSet/GameplayAbility/GameplayEffect/CueNotify asset blueprints, but for an actor blueprint that owns an ASC (BP_GASHero, parent Character) returns bare metadata with no gasType/ASC/replicationMode/abilities — even though blueprint.scs.get confirms the AbilitySystem ASC is present. Misleading false-negative GAS readback; handler at GASHandler.cpp:2236 has no ASC-owner branch. Verified live this iteration (verbatim responses above).
- `#2-additional-ge-executions-tags` `OPEN` reporter — Additional evidence (same defect family, different gasType branch): the GameplayEffect branch of get_gas_info (GASHandler.cpp:2278-2298) emits ONLY `gasType`/`durationPolicy`/`stackingType`/`modifierCount`/`cueCount` and never serializes `EffectCDO->Executions` (the wired GameplayEffectExecutionCalculation) nor `EffectCDO->InheritableOwnedTagsContainer` (granted owned tags). A task that wires an execution calc + a granted tag onto a GameplayEffect and then asks get_gas_info to "show the execution calc is attached and the granted tag is present" cannot be confirmed via this readback — same false-negative as the ASC-owner case. Writes provably persist; only the readback omits them. Verbatim live repro (this iteration), asset `/Game/GAS/Damage/GE_FireballDamage`: `gas.get_gas_info {assetPath:"/Game/GAS/Damage/GE_FireballDamage"}` -> `{"assetPath":"/Game/GAS/Damage/GE_FireballDamage","assetName":"GE_FireballDamage","class":"Blueprint","type":"Blueprint","generatedClass":"GE_FireballDamage_C","parentClass":"GameplayEffect","gasType":"GameplayEffect","durationPolicy":0,"stackingType":0,"modifierCount":0,"cueCount":0}` (no execution-calc field, no granted-tags field). Independent proof the data persisted (so readback-only omission): `gas.add_effect_execution_calculation {blueprintPath:".../GE_FireballDamage", calculationClass:".../ExecCalc_Damage.ExecCalc_Damage_C"}` -> `{"executionCount":2}` and `gas.set_effect_tags {blueprintPath:".../GE_FireballDamage", grantedTags:["Effect.Damage.Fire"]}` -> `{"tagsAdded":["Effect.Damage.Fire"]}`. Fix (extends the proposed one): in the GameplayEffect branch also emit `executionCount` (+ per-execution `calculationClass` names) from `EffectCDO->Executions` and a `grantedTags` list from `InheritableOwnedTagsContainer` so the GAS readback surfaces the configured execution calc and owned tags.
- `#3-rescope-and-fix` `IN-REVIEW` developer — Rescoped (adversarial lens: the "granted abilities" part of the original Fix was unsatisfiable — `gas.grant_ability` only creates an empty `InitialAbilities` array var and never records a per-ability grant on the CDO, so a granted-abilities field would only ever echo an empty list; dropped that from the Fix and the title). Implemented the rest as a code fix in `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Systems/GASHandler.cpp`: (a) added a final `else` branch in `gas.get_gas_info`'s blueprint path that calls a new `CollectAbilitySystemOwnerInfo()` helper — it reuses the SCS-template + generated-class-CDO ASC detection already used by `gas.grant_ability` and, when an `UAbilitySystemComponent` is found, stamps `gasType:"AbilitySystemOwner"` plus an `abilitySystemComponents` list (each entry: `name`, `source` scs/native, `replicationMode` full/mixed/minimal read off `ASC->ReplicationMode`); (b) extended the GameplayEffect branch (#2) to emit `executionCount` + an `executionClasses` array from `EffectCDO->Executions` and a `grantedTags` array from `EffectCDO->InheritableOwnedTagsContainer.CombinedTags`. Tests added in `Private/Tests/Gameplay/TestGASHandlers.cpp` (exercise production handlers, not copies): `get_gas_info.AbilitySystemOwnerEmitsGasType` builds a Character BP, adds+configures the ASC via the real `gas.add_ability_system_component`/`gas.configure_asc` handlers, then asserts `get_gas_info` returns `gasType:"AbilitySystemOwner"` and an ASC entry with `replicationMode:"mixed"`; `get_gas_info.GameplayEffectEmitsExecutionsAndTags` wires an execution calc + a (transiently-registered) granted tag onto a GE and asserts `executionCount==1` and `grantedTags` contains the tag. Both fail if the respective branch is reverted. Not compiled/tested here (later phase).
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
