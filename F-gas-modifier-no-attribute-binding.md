---
id: F-gas-modifier-no-attribute-binding
title: "gas.add_effect_modifier cannot bind a modifier to its modified attribute (Modifiers[].Attribute always empty)"
status: IN-REVIEW
severity: High
category: feature
tags: [gas, gameplay-effect, modifier, attribute, authoring]
---

# gas.add_effect_modifier cannot bind a modifier to its modified attribute

`gas.add_effect_modifier` adds an entry to a GameplayEffect's `Modifiers[]`
array but exposes **no parameter for the attribute the modifier targets**.
Its only params are `blueprintPath`, `operation`, and `magnitude`. The handler
constructs `FGameplayModifierInfo Modifier;` and sets only `ModifierOp` and
`ModifierMagnitude` — it never touches `Modifier.Attribute` (the
`FGameplayAttribute` naming which attribute is modified). Every modifier
created therefore lands with an empty attribute, which makes it inert: a real
`UGameplayEffect` modifier with an unset `Attribute` modifies nothing.

No sibling RPC closes the gap. `gas.set_modifier_magnitude` and
`gas.set_modifier_magnitude_setbycaller` only edit `Modifiers[].ModifierMagnitude`;
no handler anywhere writes `Modifiers[].Attribute`. (Contrast
`gas.set_execution_capture`, which *does* resolve and bind an
`FGameplayAttribute` from an `AttributeSetClassPath_C.AttrName` spec — the
plumbing exists, it is just not wired into modifier authoring.)

**Impact:** A canonical, simple GAS task — author a GameplayEffect whose
additive modifier boosts AttackPower (and another that reduces Armor) — is
impossible end-to-end through MCP. The modifiers can be created with the right
op and magnitude, but cannot be pointed at AttackPower / Armor, so the effect
is non-functional. This is the bulk of what attribute-modifying GameplayEffects
do; without it the modifier surface produces only no-op modifiers.

**Note (clean error, not a bug):** passing the attribute is correctly *rejected*,
not silently dropped — `add_effect_modifier {attribute:"AttackPower"}` returns
`[UNKNOWN_PARAMS] Unknown parameter(s) for 'gas.add_effect_modifier': [attribute].
Valid parameters: [blueprintPath, operation, magnitude]`, and `attributeName`
is rejected the same way. So the gap is a genuine missing capability, not a
contract violation.

**Verbatim repro (live, GE_OracleModTest under /Game/Abilities/Effects):**

```
call gas.create_gameplay_effect {name:"GE_OracleModTest", path:"/Game/Abilities/Effects", durationType:"has_duration"}
  -> ok

call gas.add_effect_modifier {blueprintPath:".../GE_OracleModTest", operation:"additive", magnitude:25, attribute:"AttackPower"}
  -> [UNKNOWN_PARAMS] Unknown parameter(s) for 'gas.add_effect_modifier': [attribute]. Valid parameters: [blueprintPath, operation, magnitude].

call gas.add_effect_modifier {blueprintPath:".../GE_OracleModTest", operation:"additive", magnitude:25, attributeName:"AttackPower"}
  -> [UNKNOWN_PARAMS] Unknown parameter(s) for 'gas.add_effect_modifier': [attributeName]. ...

call gas.add_effect_modifier {blueprintPath:".../GE_OracleModTest", operation:"additive", magnitude:25}
  -> {"modifierCount":1}   (succeeds, but the modifier targets nothing)

asset.dump .../GE_OracleModTest -> properties.json shows Modifiers[0]:
  "Attribute": { "Attribute": {"_kind":"FFieldPathProperty","value":""},
                 "AttributeName":"", "AttributeOwner": null },
  "ModifierOp": "AddBase"
```

The dumped `Modifiers[0].Attribute` is fully empty (`AttributeName:""`,
`AttributeOwner:null`), confirming the modifier is unbound.

**Fix:** add an optional `attribute` (or `attributeName`) param to
`gas.add_effect_modifier` — and ideally a paired `gas.set_modifier_attribute`
for editing an existing modifier — that resolves an `FGameplayAttribute` the
same way `gas.set_execution_capture` does (the `AttributeSetClassPath_C.AttrName`
spec naming a compiled `FProperty` on a `UAttributeSet` subclass) and assigns it
to `FGameplayModifierInfo.Attribute`. Echo the resolved attribute back in the
result so callers can confirm the binding. The capture-resolution code in
`Handlers/Systems/GASHandler.cpp` (`gas.set_execution_capture`, around the
`FGameplayAttribute Attr(AttrProperty)` construction) is the reusable
precedent.

## History
- `#1-initial-repro` `OPEN` reporter — `gas.add_effect_modifier` exposes only `blueprintPath`/`operation`/`magnitude` and the handler never sets `FGameplayModifierInfo.Attribute`, so every modifier it creates has an empty `Attribute` and is inert; no sibling RPC (`set_modifier_magnitude`, `set_modifier_magnitude_setbycaller`) writes the attribute either. Replay-confirmed live: `attribute`/`attributeName` are cleanly rejected with `UNKNOWN_PARAMS`; a plain add succeeds and `asset.dump` shows `Modifiers[0].Attribute` empty (`AttributeName:""`, `AttributeOwner:null`). Distinct from `F-gas-effect-period-and-input-binding` (period/input/SetByCaller/execution-capture — does not cover modifier attribute binding). Proposes an optional `attribute` param on `add_effect_modifier` (plus a `set_modifier_attribute` peer) resolving `FGameplayAttribute` via the `AttributeSetClassPath_C.AttrName` spec like `gas.set_execution_capture`.
- `#2-implemented` `IN-REVIEW` developer — Implemented the root-cause fix in `Handlers/Systems/GASHandler.cpp`. (1) Added a shared static helper `ResolveGameplayAttributeFromSpec(AttrSpec, OutAttr, OutErrCode, OutErrMsg)` that resolves an `FGameplayAttribute` from the `AttributeSetClassPath_C.AttrName` spec (split-on-last-dot → `LoadObject<UClass>` → `FindPropertyByName` → `FGameplayAttribute(Prop)`), factored out of `gas.set_execution_capture` (now calls the helper, no behavior change). (2) `gas.add_effect_modifier` gained an optional `attribute` param (alias `attributeName`, registered on the ParamSpec so the dispatcher accepts it); when supplied it binds `Modifier.Attribute` and echoes the resolved attribute name back; omitting it preserves legacy unbound behavior; a bad spec is rejected with `ATTRIBUTE_NOT_FOUND`/`ATTRIBUTE_SET_NOT_FOUND` instead of producing an inert modifier. (3) Added a paired `gas.set_modifier_attribute` handler (required `attribute`, `modifierIndex`) to rebind an existing modifier. Wiki: added a "binding the modified attribute" section to `docs/wiki-src/gas.md`. Tests: behavioral end-to-end in `Tests/Gameplay/TestGASHandlers.cpp` (`gas.add_effect_modifier.BindsAttribute`) builds a real AttributeSet (AttackPower) + GE, binds the modifier, and asserts the live CDO `Modifiers[0].Attribute.IsValid()` and `.GetName()=="AttackPower"`, rejects a bogus attribute, and rebinds via `set_modifier_attribute` — reverting the `Modifier.Attribute` assignment fails it; plus GAS-agnostic registration regressions in `Tests/EditorOps/TestSystemsHandlers.cpp` (`gas.add_effect_modifier.ExposesAttributeParam`, `gas.set_modifier_attribute.Registered`) that fail if the param spec or handler is reverted. Not yet compiled/run (later phase verifies green).
- `#3-additional-broken-doc-example` `OPEN` reporter — Additional evidence (the shipped param-description example is verbatim-unloadable; not yet caught by this IN-REVIEW fix). The new `attribute` param on `gas.add_effect_modifier` is described (`GASHandler.cpp:1660`, and identically on `gas.set_modifier_attribute` at `:1792`) as: `"Attribute this modifier targets, in 'AttributeSetClassPath_C.AttrName' form (e.g. '/Game/GAS/BP_MyAttributeSet_C.AttackPower')"`. That documented form is structurally wrong and **cannot resolve**: `ResolveGameplayAttributeFromSpec` splits on the last `.` and feeds the left side straight to `LoadObject<UClass>`, but a generated-class object path is `<Package>.<Class>_C` (with a `.` before `_C`), never `<Package>_C`. So copy-pasting the documented `'/Game/X/BP_MyAttributeSet_C.AttackPower'` form always yields `[ATTRIBUTE_SET_NOT_FOUND]`. The correct form is the package-qualified `<Package>.<Class>_C.<AttrName>` (which the `docs/wiki-src/gas.md` overlay added in `#2` actually uses: `/Game/GAS/AS_Combat.AS_Combat_C.AttackPower`) — so the overlay example is right but the param-description example string still ships the broken shape. Replay-confirmed live (REALISM mode, action-RPG GAS-setup task): created `/Game/ReplayGAS/ReplayAttrSet` (AttributeSet, attr `Health`) + `/Game/ReplayGAS/ReplayGE` (GameplayEffect), then `gas.add_effect_modifier {blueprintPath:".../ReplayGE", magnitude:-10, operation:"additive", attribute:"/Game/ReplayGAS/ReplayAttrSet_C.Health"}` (the documented `_C.AttrName` form) → `[ATTRIBUTE_SET_NOT_FOUND] Could not load attribute set class '/Game/ReplayGAS/ReplayAttrSet_C'`; the same call with the package-qualified `attribute:"/Game/ReplayGAS/ReplayAttrSet.ReplayAttrSet_C.Health"` → success (`{modifierCount:1, attribute:"Health"}`). The attempt agent that filed this hit the identical first-try failure and recovered only by switching to the package-qualified form. Fix as part of closing this ticket: correct the example string in both param descriptions (`GASHandler.cpp:1660` and `:1792`) to the package-qualified `<Package>.<Class>_C.<AttrName>` form, matching the overlay; same defect class as `E-add-mapping-example-wrong-param` (a verbatim-broken documented example).
- `#4-recurrence-broken-doc-example` `IN-REVIEW` reporter — Process-audit recurrence of the `#3` broken-example friction (a separate REALISM task; aggregating, not re-filing). Action-RPG GAS setup under `/Game/RPG/GAS`: after creating `PlayerAttributeSet` (Health/MaxHealth/Mana) the author's first `gas.add_effect_modifier` for GE_Burning used the documented `_C.AttrName` shape and failed — `[ATTRIBUTE_SET_NOT_FOUND] Could not load attribute set class '/Game/RPG/GAS/PlayerAttributeSet_C'` — then recovered by inserting a `gas.get_gas_info` lookup to learn the generated-class name and switching to the package-qualified `/Game/RPG/GAS/PlayerAttributeSet.PlayerAttributeSet_C.Health` form, which succeeded. Friction note (verbatim): "gas.add_effect_modifier's attribute format in the wiki (\"/Game/GAS/BP_MyAttributeSet_C.AttackPower\") is misleading — it omits the required Package.ClassName form, so my first bind failed ATTRIBUTE_SET_NOT_FOUND until I used \"/Game/RPG/GAS/PlayerAttributeSet.PlayerAttributeSet_C.Health\"; the wiki should show the package-qualified object path." The author also notes a last-resort dive into `GASHandler.cpp` to learn the attribute-path split logic — a discoverability gap the doc fix already scoped in `#3` (correct the example string at `GASHandler.cpp:1660` and `:1792` to `<Package>.<Class>_C.<AttrName>`) would close. Recovery cost this task: 1 failed bind + 1 extra `gas.get_gas_info` probe + 1 corrected retry. Confirms `#3`'s example-string fix should also propagate into the `docs/wiki-src/gas.md` overlay's `add_effect_modifier` example so wiki readers (not just param-description readers) get the package-qualified form.
