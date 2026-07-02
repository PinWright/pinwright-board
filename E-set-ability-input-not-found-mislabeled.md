---
id: E-set-ability-input-not-found-mislabeled
title: "gas.set_ability_input reports PROPERTY_NOT_INT (a type-mismatch code) when AbilityInputID is absent, and the wiki never says an MCP-created ability lacks it"
status: OPEN
severity: Medium
category: ergonomic
tags: [error-code-mislabels-cause, gas, set-ability-input, undocumented-precondition]
encounters: 1
lastSeen: 2026-07-02T06:05:09.4953018+03:00
---

# `gas.set_ability_input` mislabels a missing `AbilityInputID` property as `PROPERTY_NOT_INT`

`gas.set_ability_input` (cdo mode, the default) writes an int/byte value onto a
**pre-existing** property named `AbilityInputID` on the ability CDO — it does not
create that property. A vanilla `UGameplayAbility` has no `AbilityInputID`
member, and `gas.create_gameplay_ability` produces exactly a vanilla-parented
ability (`parentClass: GameplayAbility`). So the plugin's own ability-creation
path yields an ability that its own input-binding path cannot bind, until the
caller manually adds an int variable via `blueprint.add_variable` + `blueprint.compile`.

Two ergonomic defects compound on that missing-property path:

1. **The error code contradicts the real cause.** When the property is absent
   the handler emits code `PROPERTY_NOT_INT` — a code whose name says "the
   property exists but is the wrong type". The message body says the opposite
   ("not found on ability class"). The *same* code is also emitted for the
   genuine type-mismatch branch. A caller who dispatches on the error code (the
   machine-readable signal) is told to fix a type when the real fix is to *add*
   the property. It should be a distinct code such as `PROPERTY_NOT_FOUND`.

2. **The precondition is undocumented.** The `gas.set_ability_input` wiki page
   lists `propertyName` (default `AbilityInputID`) but never states that the
   property must already exist on the ability class, nor that an ability made by
   `gas.create_gameplay_ability` won't have one. With no hint, discovering the
   `blueprint.add_variable` workaround required a plugin-source dive into
   `GASHandler.cpp` (the attempt agent flagged this as a "last resort").

This is the same shape as `E-widget-bind-event-misleading-name` (a tool that
requires a pre-existing artifact it won't create) — but that one's error text
was *helpful*; here the error **code** actively misdirects. It is a downstream
ergonomic wrinkle of the (DONE) feature `F-gas-effect-period-and-input-binding`,
which by design writes to a pre-existing `AbilityInputID` (the Lyra/ARPG
convention); the feature works when the property is present, so this is not a
regression of that feature but a new ergonomic gap in how it reports failure and
composes with `gas.create_gameplay_ability`.

## Verbatim repro (live, replay-confirmed via `mcp__pinwright__call`)

1. `gas.create_gameplay_ability` `{name:"GA_ReplayInputTest", path:"/Game/GAS/Abilities"}`
   → `{"assetPath":"/Game/GAS/Abilities/GA_ReplayInputTest","name":"GA_ReplayInputTest","parentClass":"GameplayAbility"}`
   (vanilla parent — no `AbilityInputID` member).
2. `gas.set_ability_input` `{blueprintPath:"/Game/GAS/Abilities/GA_ReplayInputTest", inputIDValue:1}`
   → **`[PROPERTY_NOT_INT] Property 'AbilityInputID' not found on ability class`**

The property genuinely does not exist, so the rejection is correct — but the
code `PROPERTY_NOT_INT` names a type mismatch on an existing property, not an
absent one. (Replay asset deleted via `asset.delete` afterward.)

## Guilty source

`Plugins/PinWright/Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`:

```
3679        FProperty* Prop = Ability->GetClass()->FindPropertyByName(FName(*PropertyName));
3680        if (!Prop)
3681        {
3682            Ctx.SendError(TEXT("PROPERTY_NOT_INT"), FString::Printf(TEXT("Property '%s' not found on ability class"), *PropertyName));
3683            return true;
3684        }
```

The genuine type-mismatch branch reuses the identical code:

```
3695            Ctx.SendError(TEXT("PROPERTY_NOT_INT"), FString::Printf(TEXT("Property '%s' is not an int or byte property"), *PropertyName));
```

## Impact

An agent authoring an input-bound ability entirely through MCP hits a
misdirecting failure on the default path: the error code tells it the property
is the wrong type (implying "change the type"), when in fact it must first
create the property. With no wiki hint about the precondition, diagnosing this
took a plugin-source dive. The write itself is correctly refused, and a
workaround exists (`blueprint.add_variable AbilityInputID:int` + `blueprint.compile`,
then retry), so this is a blocker-with-workaround rather than a hard gap.

severity rationale: impact=blocker-with-workaround (source-dive to diagnose) × reach=every MCP-authored input-bound ability (within GAS authoring) -> Medium

## Fix

1. Split the error code: emit `PROPERTY_NOT_FOUND` for the `!Prop` branch
   (line 3682), keep `PROPERTY_NOT_INT` only for the actual not-int/byte branch
   (line 3695).
2. Document the precondition on the `gas.set_ability_input` wiki page: cdo mode
   writes an existing int/byte property (default `AbilityInputID`); abilities
   created by `gas.create_gameplay_ability` do not have one — add it first via
   `blueprint.add_variable` (or parent the ability to a class that declares it).
   Optionally, have the handler auto-create the int property (or point the error
   at `blueprint.add_variable`) so the cdo path composes with the plugin's own
   ability-creation path in one step.

## History
- `#1-initial-repro` `OPEN` reporter — `gas.set_ability_input {blueprintPath:<vanilla GA>, inputIDValue:1}` on an ability freshly made by `gas.create_gameplay_ability` (parentClass GameplayAbility, no `AbilityInputID` member) returns `[PROPERTY_NOT_INT] Property 'AbilityInputID' not found on ability class`. Replay-confirmed live against `mcp__pinwright__call`. Two ergonomic defects: (a) the error CODE `PROPERTY_NOT_INT` names a type mismatch but the property is absent — the same code is reused for both the not-found branch (`GASHandler.cpp:3682`) and the genuine not-int branch (`:3695`); should be a distinct `PROPERTY_NOT_FOUND`. (b) The wiki never states cdo mode needs a pre-existing int/byte property, nor that MCP-created abilities lack it, forcing a plugin-source dive to find the `blueprint.add_variable`+`compile` workaround. Not a regression of the DONE feature `F-gas-effect-period-and-input-binding` (which writes to an existing property by design), but a new ergonomic gap in its failure reporting and composition with `gas.create_gameplay_ability`. Replay asset cleaned up via `asset.delete`.
