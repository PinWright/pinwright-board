---
id: B-set-ability-cooldown-cost-class-path-silent-drop
title: "gas.set_ability_cooldown / set_ability_costs silently no-op (ok:true) when the GameplayEffect path doesn't resolve as a UClass — only the .._C generated-class path persists"
status: IN-REVIEW
severity: High
category: bug
tags: [gas, set_ability_cooldown, set_ability_costs, silent-noop, loadclass, cdo, path-resolution]
---

# `gas.set_ability_cooldown` / `gas.set_ability_costs` report success while writing nothing to the CDO

Both verbs resolve their effect path with `LoadClass<UGameplayEffect>` and gate
the write on a bare `if (Class) { ... }` with **no else-branch**, then fall
through to `Ctx.SendSuccess` regardless:

`GASHandler.cpp` `gas.set_ability_cooldown` (1058-1065):
```cpp
if (!CooldownEffectPath.IsEmpty())
{
    UClass* CooldownClass = LoadClass<UGameplayEffect>(nullptr, *CooldownEffectPath);
    if (CooldownClass)
    {
        SetAbilityPropertyValue(AbilityCDO, FName(TEXT("CooldownGameplayEffectClass")), TSubclassOf<UGameplayEffect>(CooldownClass));
    }
}
// ...no else, always:
Ctx.SendSuccess(Result);   // echoes back cooldownEffectPath as if it took
```

`gas.set_ability_costs` (1005-1012) is byte-for-byte the same shape with
`CostGameplayEffectClass`.

`LoadClass<UGameplayEffect>` requires a **class** object path
(`/Game/.../GE_X.GE_X_C`). When the caller passes the **plain blueprint-asset
path** (`/Game/.../GE_X`) — which the wiki's own wording invites ("`cooldownEffectPath`:
Path to the cooldown GameplayEffect class") and which every other GAS verb in a
normal authoring run accepts as an asset path (`gas.set_effect_duration`,
`gas.add_effect_modifier`, `gas.get_gas_info`, `asset.dump`) — `LoadClass`
returns null, the guard skips the write, and the handler **still returns success
and echoes the path back verbatim**. The cooldown/cost is never attached; the
CDO field stays `null`. There is no error, no warning, no `applied:false`, no
hint that the path needed a `.._C` suffix — the response is indistinguishable
from a real write.

This is a silent success-with-no-effect on a write RPC: the single worst failure
mode for an automation agent, because the documented readback can't see it
(`get_gas_info` omits cooldown/cost — see `E-gas-info-...`), so the agent
believes the ability is wired and hands off a broken asset. The only way to
discover the drop is an out-of-band `property.get` on the CDO field, which a
caller has no reason to do after a clean success.

## Replay-confirmed (live, this session — focus `gas.set_ability_cooldown`)

Built a minimal repro on the live editor and read the CDO back via `property.get`:

1. `asset.create_folder {path:"/Game/OracleProbe/AbilityKit"}` → ok
2. `gas.create_gameplay_ability {path:"/Game/OracleProbe/AbilityKit", name:"GA_Probe"}` → ok
3. `gas.create_gameplay_effect {path:"/Game/OracleProbe/AbilityKit", name:"GE_ProbeCooldown", durationType:"has_duration"}` → ok
4. **plain asset path** —
   `gas.set_ability_cooldown {blueprintPath:"/Game/OracleProbe/AbilityKit/GA_Probe", cooldownEffectPath:"/Game/OracleProbe/AbilityKit/GE_ProbeCooldown"}`
   → SUCCESS, echoed
   `{"blueprintPath":"/Game/OracleProbe/AbilityKit/GA_Probe","cooldownEffectPath":"/Game/OracleProbe/AbilityKit/GE_ProbeCooldown"}`
   (looks applied)
5. readback —
   `property.get {objectPath:"/Game/OracleProbe/AbilityKit/GA_Probe.Default__GA_Probe_C", propertyName:"CooldownGameplayEffectClass"}`
   → `{"propertyName":"CooldownGameplayEffectClass","value":null,...}` — **never written**
6. **`.._C` class path** —
   `gas.set_ability_cooldown {... cooldownEffectPath:"/Game/OracleProbe/AbilityKit/GE_ProbeCooldown.GE_ProbeCooldown_C"}`
   → SUCCESS
7. readback → `value:"/Game/OracleProbe/AbilityKit/GE_ProbeCooldown.GE_ProbeCooldown_C"` — **persisted**

So step 4 (plain asset path) and step 6 (`.._C` class path) return the **same
clean success**, but only step 6 actually writes. `gas.set_ability_costs` shares
the identical `LoadClass` + bare-`if` code path (`GASHandler.cpp:1005-1012`,
`CostGameplayEffectClass`) and the attempt agent observed the same null-readback
on it for the plain path; both should be fixed together.

## What it should do / fix

Resolve the effect path the same forgiving way the rest of the gateway treats
class-path inputs, and never silent-success on a non-resolving path. Concretely:

- Route the effect path through the **canonical `ResolveUClass` resolver**
  (`Utils/ClassUtils.cpp`) instead of the raw `LoadClass<UGameplayEffect>`.
  `ResolveUClass` already does the forgiving resolution this ticket asks for —
  direct find/load, self-append `_C`, then `LoadObject<UBlueprint>` →
  `GeneratedClass` — so a plain asset path (`/Game/.../GE_X`) and the
  generated-class path (`/Game/.../GE_X.GE_X_C`) both resolve. Do **not** hand-roll
  the BP/`GeneratedClass`/`_C` fallback inline; `ResolveUClass` is the resolver
  34 source files already share (it was built by `E-class-name-format-inconsistency`,
  DONE).
- After resolving, guard with `IsChildOf(UGameplayEffect::StaticClass())`. If the
  path resolves to no `UClass` → reject with `NOT_FOUND`; if it resolves to a
  non-GE class → reject with `INVALID_TYPE`. Do **not** fall through to
  `SendSuccess`. This mirrors `E-create-hud-requires-c-suffix-class-path`
  (IN-REVIEW), which fixed the identical raw-`LoadClass` `_C`-suffix defect on
  `ui.create_hud` by routing `widgetPath` through `ResolveUClass` + an
  `IsChildOf(UUserWidget)` guard, and the validate-before-mutate convention
  adopted for the GAS tag-write family (`E-gas-set-effect-tags-drops-unregistered`).
- Apply the identical fix to both `gas.set_ability_cooldown` (`cooldownEffectPath`)
  and `gas.set_ability_costs` (`costEffectPath`).

(Then update the `gas.md` overlay so the param doc no longer implies a bare
asset path works — or, preferably, make the bare asset path work as above so the
doc is already correct.)

## Distinct from existing GAS tickets

- `E-gas-set-effect-tags-drops-unregistered` — silent-drop on the **tag-write**
  family (unregistered gameplay tags), a different input class and different
  handlers; this is the **effect-class path** resolution on the cooldown/cost
  setters.
- `E-gas-info-effect-modifiers-duration-value-thin` / `E-gas-info-skips-asc-owner-actor`
  — readback thinness in `get_gas_info`; this is a write-side silent no-op.
- `F-gas-modifier-no-attribute-binding`, `F-gas-effect-period-and-input-binding`
  — capability gaps on the modifier/effect verbs, not the ability cooldown/cost
  setters. No existing file mentions `set_ability_cooldown`/`set_ability_costs`.

## History
- `#2-reword-and-fix` `IN-REVIEW` developer — Rewrote the **Fix** to steer through the canonical `ResolveUClass` resolver (the in-tree forgiving resolver, not a hand-rolled BP/`GeneratedClass`/`_C` fallback), per `E-create-hud-requires-c-suffix-class-path`. Implemented: in `Source/EditorAutomationRpcGateway/Private/Handlers/Systems/GASHandler.cpp`, replaced the raw `LoadClass<UGameplayEffect>` + bare-`if` + unconditional `SendSuccess` in both `gas.set_ability_costs` (CostGameplayEffectClass) and `gas.set_ability_cooldown` (CooldownGameplayEffectClass) with `ResolveUClass(path)` → `IsChildOf(UGameplayEffect)` guard, rejecting `NOT_FOUND` (no class resolves) / `INVALID_TYPE` (resolves to a non-GE class) instead of silent-success. `ResolveUClass` is reachable in this TU via the `EditorAutomationRpcGatewayHelpers.h` umbrella (`Utils/ClassUtils.h`); no new include needed. Regression test `EditorAutomationRpcGateway.gas.set_ability_cooldown_costs.PlainPathPersistsAndRejectsMissing` added to `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestGASHandlers.cpp`: creates an ability + cooldown/cost GEs, calls both setters with the PLAIN asset path, asserts both classes are persisted on the ability CDO (reads `Cooldown/CostGameplayEffectClass` via `FClassProperty`), and asserts a non-resolving path is rejected `NOT_FOUND` on both verbs. Reverting the fix fails this test two ways (null readback on the plain path; ok:true on the bogus path). Not compiled/tested here — a later phase drives green.
- `#1-initial-repro` `OPEN` reporter — Seed `gas.set_ability_cooldown` (Fireball ability-kit task). Replay-confirmed live: with the plain blueprint-asset path `/Game/OracleProbe/AbilityKit/GE_ProbeCooldown`, `gas.set_ability_cooldown` returns clean success and echoes the path, but `property.get` on `Default__GA_Probe_C.CooldownGameplayEffectClass` reads `null` — the write was silently skipped because `LoadClass<UGameplayEffect>` only resolves the `.._C` class path; re-running with `…/GE_ProbeCooldown.GE_ProbeCooldown_C` persisted correctly. Root cause: `GASHandler.cpp:1058-1065` (cooldown) and `:1005-1012` (costs) gate the write on `if (LoadClass(...))` with no else and unconditionally `SendSuccess`. Sibling `gas.set_ability_costs` has the identical defect (`CostGameplayEffectClass`, observed null-readback by the attempt agent). Silent success-with-no-effect, masked downstream by `get_gas_info` omitting cooldown/cost fields. Proposed fix: fall back to `LoadObject<UBlueprint>->GeneratedClass`/append `_C`, and reject (not silent-success) when the path resolves to no `UGameplayEffect` class; apply to both setters.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
