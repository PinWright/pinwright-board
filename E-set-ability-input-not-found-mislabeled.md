---
id: E-set-ability-input-not-found-mislabeled
title: "gas.set_ability_input reports PROPERTY_NOT_INT (a type-mismatch code) when AbilityInputID is absent, and the wiki never says an MCP-created ability lacks it"
status: IN-REVIEW
severity: Low
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

`Plugins/PinWright/Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`
(line numbers corrected — the reporter's original 3679/3682/3695 had drifted
~77 lines). The property-absent branch, **pre-fix**:

```
3756        FProperty* Prop = Ability->GetClass()->FindPropertyByName(FName(*PropertyName));
3757        if (!Prop)
3758        {
3759            Ctx.SendError(TEXT("PROPERTY_NOT_INT"), FString::Printf(TEXT("Property '%s' not found on ability class"), *PropertyName));
3760            return true;
3761        }
```

The genuine type-mismatch branch reuses the identical code (kept as-is by the fix):

```
3772            Ctx.SendError(TEXT("PROPERTY_NOT_INT"), FString::Printf(TEXT("Property '%s' is not an int or byte property"), *PropertyName));
```

A distinct-code sibling already exists in the same function — the InputAction
branch emits `INPUT_ACTION_PROPERTY_NOT_FOUND` (`GASHandler.cpp:3804`) for its
own absent-property case — so `PROPERTY_NOT_INT` on the absent `!Prop` branch was
an internal inconsistency, not a deliberate choice.

## Impact

An agent authoring an input-bound ability entirely through MCP hits a
misdirecting failure on the default path: the error code tells it the property
is the wrong type (implying "change the type"), when in fact it must first
create the property. With no wiki hint about the precondition, diagnosing this
took a plugin-source dive. The write itself is correctly refused, the message
BODY already names the true cause ("not found on ability class"), and a
workaround exists (`blueprint.add_variable AbilityInputID:int` + `blueprint.compile`,
then retry), so this is pure friction rather than a blocker.

severity rationale: the two actionable defects are a misnamed error CODE (naming)
and a missing precondition note (docs, discoverability) — both Low "pure friction"
per the board rubric. The write is correctly refused (no false-success, no data
loss) and the message body already names the true cause, so it is friction, not a
soft blocker requiring a workaround to *succeed*. `set_ability_input`'s cdo path is
a specialized GAS-authoring surface, not an every-session method, so no upward
reach bump -> Low

## Fix (shipped — IN-REVIEW)

1. **Split the error code** (done): the `!Prop` branch now emits
   `PROPERTY_NOT_FOUND` (`GASHandler.cpp:3759`); the genuine not-int/byte branch
   keeps `PROPERTY_NOT_INT` (`:3772`). Reuses the plugin-wide `PROPERTY_NOT_FOUND`
   convention (`ErrorCodes.h:463`) and matches the sibling
   `INPUT_ACTION_PROPERTY_NOT_FOUND` already emitted for the absent InputAction
   property in the same function (`:3804`).
2. **Document the precondition** (done): the `propertyName` param spec
   (`GASHandler.cpp:3677`) now states the property must already exist on the
   ability class, that `gas.create_gameplay_ability` does not add one (add it via
   `blueprint.add_variable` first), and which code each failure returns. This is
   the tool's self-documenting surface — there is no separate `wiki-src` overlay
   file for gas, so the param spec is where the precondition belongs.
3. **Auto-create the int property — declined** (gold-plate). Same shape as the
   DONE sibling `E-widget-bind-event-misleading-name`, whose shipped disposition
   fixed the reporting and explicitly declined auto-creation; a write verb should
   not silently mutate the ability class schema.

## History
- `#1-initial-repro` `OPEN` reporter — `gas.set_ability_input {blueprintPath:<vanilla GA>, inputIDValue:1}` on an ability freshly made by `gas.create_gameplay_ability` (parentClass GameplayAbility, no `AbilityInputID` member) returns `[PROPERTY_NOT_INT] Property 'AbilityInputID' not found on ability class`. Replay-confirmed live against `mcp__pinwright__call`. Two ergonomic defects: (a) the error CODE `PROPERTY_NOT_INT` names a type mismatch but the property is absent — the same code is reused for both the not-found branch (`GASHandler.cpp:3682`) and the genuine not-int branch (`:3695`); should be a distinct `PROPERTY_NOT_FOUND`. (b) The wiki never states cdo mode needs a pre-existing int/byte property, nor that MCP-created abilities lack it, forcing a plugin-source dive to find the `blueprint.add_variable`+`compile` workaround. Not a regression of the DONE feature `F-gas-effect-period-and-input-binding` (which writes to an existing property by design), but a new ergonomic gap in its failure reporting and composition with `gas.create_gameplay_ability`. Replay asset cleaned up via `asset.delete`.
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORD then fix. Verified the defect in current source: `GASHandler.cpp:3759` (the `!Prop` absent-property branch) emitted `PROPERTY_NOT_INT`, identical to the genuine wrong-type branch at `:3772`, while the sibling absent-InputAction branch already used `INPUT_ACTION_PROPERTY_NOT_FOUND` (`:3804`) — an internal inconsistency. Reword: corrected the stale line citations (reporter's 3679/3682/3695 -> actual 3756/3759/3772) and dropped severity Medium -> Low (a misnamed error CODE + a missing precondition note are both Low "pure friction" per the board rubric; the write is correctly refused and the message body already names the true cause, so it is friction not a soft blocker; rare GAS-authoring path -> no reach bump). Fix: split the code — `:3759` now emits `PROPERTY_NOT_FOUND` (reusing the plugin-wide convention, `ErrorCodes.h:463`), `:3772` keeps `PROPERTY_NOT_INT`; and documented the precondition in the `propertyName` param spec (`:3677`). Declined the optional auto-create (gold-plate, per the DONE sibling `E-widget-bind-event-misleading-name` which fixed reporting and declined auto-creation). Files: `Plugins/PinWright/Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`, `Plugins/PinWright/Source/PinWright/Private/Tests/Gameplay/TestGASHandlers.cpp`. Regression test `PinWright.gas.set_ability_input.MissingPropertyDistinctFromWrongType`: builds a vanilla ability in-code via `gas.create_gameplay_ability` (no `AbilityInputID` member), asserts the absent-property call returns `PROPERTY_NOT_FOUND` and a present-but-bool property (`bReplicateInputDirectly`) still returns `PROPERTY_NOT_INT` — proving the two codes are now distinct; reverting `:3759` fails the first assertion.
