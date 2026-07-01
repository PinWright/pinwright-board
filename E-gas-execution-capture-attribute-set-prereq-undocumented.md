---
id: E-gas-execution-capture-attribute-set-prereq-undocumented
title: "gas.add_attribute must compile so set_execution_capture can resolve the attribute (plus document the _C capture format)"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [gas, set_execution_capture, add_attribute, docs, discovery]
---

# `gas.add_attribute` should compile its AttributeSet, and `set_execution_capture`'s capture format needs documenting

**Primary fix (code):** `gas.add_attribute` does not compile the AttributeSet
blueprint, so the attribute never lands on the generated class — every sibling
GAS mutator (`gas.set_attribute_clamping` at `GASHandler.cpp:558`, and others)
compiles, making `add_attribute` the lone exception. The result is a hidden,
load-bearing `blueprint.compile` step that callers must discover by reading
source. Make `add_attribute` compile its blueprint (parity with
`set_attribute_clamping`) so the property is immediately visible to
`gas.set_execution_capture` and `gas.set_attribute_base_value`. Also drop the
phantom `backingPropertyName?` field advertised in `set_execution_capture`'s
param description (`GASHandler.cpp:3066`) — it is never read by the handler and
falsely implies a pure-Blueprint capture path that does not exist. Docs are the
floor on top of the code fix, not the whole remedy.

## Background

`gas.set_execution_capture` populates `RelevantAttributesToCapture` on a
`UGameplayEffectExecutionCalculation` blueprint. The wiki/method signature
presents each capture as a simple `{attribute, source, snapshot}` object, but
the handler actually requires two things the docs never mention, both of which
have to be discovered by reading `GASHandler.cpp`:

1. **`attribute` must resolve to a real property on a real AttributeSet.** The
   string is not a free-form name — it must name an actual `FProperty` on a
   `UAttributeSet` subclass (resolved via the attribute-set generated class,
   `<AttributeSetClassPath>.<Class>_C` → `AttrName`). There is no such thing
   as a pure-BP capture that materializes the backing property "under the hood"
   for the execution calc; a backing AttributeSet asset with the named
   attributes must already exist before the capture call.

2. **The AttributeSet must be COMPILED before the attributes are visible.**
   `gas.add_attribute` writes the attribute onto the AttributeSet blueprint but
   does **not** compile it, so the generated class has no property yet. Calling
   `gas.set_execution_capture` immediately after `add_attribute` fails with a
   misleading-looking error:

   `[ATTRIBUTE_NOT_FOUND] Attribute set '/Game/GAS/Damage/AS_Combat.AS_Combat_C' has no property 'AttackPower'`

   — even though `add_attribute` acked success. The fix is a manual
   `blueprint.compile` on the AttributeSet to materialize the properties on the
   generated class; only then does the same `set_execution_capture` call
   succeed (`captureCount:3`). Nothing in the wiki hints that `add_attribute`
   needs a compile pass, or that the `ATTRIBUTE_NOT_FOUND` error specifically
   means "the property exists on the blueprint but not yet on the compiled
   generated class."

The end-to-end "author an execution calc that captures source/target
attributes" intent therefore takes a buried sequence:
`create_attribute_set` → N× `add_attribute` → **`blueprint.compile`** (the
undocumented load-bearing step) → `set_execution_capture` with `_C`-form
attribute paths. A caller following the documented `{attribute,source,snapshot}`
shape against attributes they just added hits ATTRIBUTE_NOT_FOUND and has no
way to know a compile is missing.

## Evidence (this task — focus `gas.add_effect_execution_calculation`)

Friction note (verbatim): "(1) gas.set_execution_capture's wiki only says
`{attribute,source,snapshot}` — it actually requires attribute in
`AttributeSetClassPath.AttrName` _C form bound to a REAL AttributeSet property,
undocumented; had to read GASHandler.cpp and pre-create+populate an
AttributeSet. (2) Even then it failed ATTRIBUTE_NOT_FOUND until I manually
blueprint.compile'd the AttributeSet (add_attribute doesn't compile, so
properties weren't on the generated class) — no wiki hint."

Call-log cost: one failed `gas.set_execution_capture`
(`[ATTRIBUTE_NOT_FOUND] … has no property 'AttackPower'`) plus an inserted
`blueprint.compile AS_Combat saveAfterCompile=true` before the retry succeeded
(`captureCount:3`) — i.e. a wasted call + a remediation call + a source-code
read, all to recover the undocumented prerequisite. The author also had to
pre-create an `AS_Combat` AttributeSet and add three attributes (work the
documented signature gives no hint is required).

**Fix (code first, docs as the floor):**

1. **`gas.add_attribute` compiles its AttributeSet** (`GASHandler.cpp`) — after
   `AddMemberVariable` + `MarkBlueprintAsStructurallyModified`, call
   `FKismetEditorUtilities::CompileBlueprint` (then `McpSafeAssetSave`), exactly
   as the sibling `gas.set_attribute_clamping` does. This eliminates the hidden
   `blueprint.compile` step: the attribute is on the generated class as soon as
   `add_attribute` acks, so `set_execution_capture` and `set_attribute_base_value`
   resolve it immediately.
2. **Drop the phantom `backingPropertyName?`** from `set_execution_capture`'s
   `captures` param description (`GASHandler.cpp:3066`) — it is never read and
   falsely advertises a pure-BP path; replace it with the real `_C`-form contract.
3. **Sharpen `ATTRIBUTE_NOT_FOUND`** in `set_execution_capture` to say the
   property is not on the **compiled generated class** and that the remedy is
   `blueprint.compile` on the AttributeSet (covers AttributeSets authored by
   means other than `gas.add_attribute`).
4. **Docs floor:** `docs/wiki-src/gas.md` — add an "Execution-calc captures"
   section stating each `attribute` must be `<AttributeSetClassPath>_C.<AttrName>`
   naming a real property on a compiled AttributeSet, the authoring order
   (`create_attribute_set → add_attribute → create_execution_calculation →
   set_execution_capture`, with `add_attribute` now compiling for you), and what
   `ATTRIBUTE_NOT_FOUND` means.

## History
- `#2-add-attribute-compiles` `IN-REVIEW` developer — Rewrote the ticket (REWORD): the root cause is a code anomaly, not just a docs gap, so the code fix is now primary with docs as the floor. Implemented in `Source/EditorAutomationRpcGateway/Private/Handlers/Systems/GASHandler.cpp`: (1) `gas.add_attribute` now calls `FKismetEditorUtilities::CompileBlueprint` + `McpSafeAssetSave` after `AddMemberVariable`/`MarkBlueprintAsStructurallyModified`, giving it parity with the sibling `gas.set_attribute_clamping` (`:558`) so the attribute lands on the generated class immediately — no hidden `blueprint.compile` step; (2) removed the phantom `backingPropertyName?` from `gas.set_execution_capture`'s `captures` param description (`:3066`, never read) and replaced it with the real `_C`-form contract; (3) sharpened the `ATTRIBUTE_NOT_FOUND` message to point at the uncompiled-generated-class root cause and the `blueprint.compile` remedy. Docs floor: added an "Execution-calc captures" section to `Docs/wiki-src/gas.md`. Regression test `FGasAddAttributeCompilesGeneratedClassTest` (`EditorAutomationRpcGateway.gas.add_attribute.CompilesGeneratedClass` in `Private/Tests/Gameplay/TestGASHandlers.cpp`) runs `create_attribute_set → add_attribute` and asserts `AttackPower` is a compiled `FProperty` on the generated class with no manual compile — it fails if the new `CompileBlueprint` call is reverted.
- `#1-initial-audit` `OPEN` reporter — Process/docs audit of the `gas.add_effect_execution_calculation` task. `gas.set_execution_capture` is documented as `{attribute,source,snapshot}` but actually requires the attribute to name a real, **compiled** AttributeSet property in `<AttributeSetPath>.<Class>_C` form; `gas.add_attribute` does not compile, so a manual `blueprint.compile` is a hidden load-bearing step. The author burned a failed `set_execution_capture` (`[ATTRIBUTE_NOT_FOUND] … has no property 'AttackPower'`), a remediation `blueprint.compile`, and a `GASHandler.cpp` source read before the capture resolved (`captureCount:3`). The `docs/wiki-src/gas.md` overlay has no execution-calc capture section and no mention of the add_attribute→compile prerequisite. Distinct from the readback gap filed as `E-gas-info-skips-asc-owner-actor` (#2) and from the feature ticket `F-gas-effect-period-and-input-binding` (which added the method but whose proposal wrongly implied pure-BP captures need no backing property). Proposes documenting the capture format + required compile order on `docs/wiki-src/gas.md`.
