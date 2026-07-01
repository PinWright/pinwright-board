---
id: B-property-reset-cdo-uses-self-not-archetype
title: "property.reset on a Blueprint CDO uses the CDO's own value as the default (no archetype/parent-class fallback), so reset is a guaranteed no-op"
status: IN-REVIEW
severity: Medium
category: bug
tags: [property, reset, cdo, archetype, default-source, blueprint]
---

# property.reset on a Blueprint CDO uses the CDO's own value as the default, not the parent-class archetype

`property.reset` documents itself as "Reset a UPROPERTY to its class default and clear explicit override metadata", and the editor's per-property reset arrow on a Blueprint CDO field reverts the property to its **archetype** value (`UObject::GetArchetype()` → the parent-class CDO). For a property that is *inherited from a parent class and not authored as a local CDO override*, that archetype value is the meaningful "default".

But when the target object is a Blueprint generated-class CDO, `property.reset` (and `property.list` / `property.get`) resolve the default as the **CDO's own current value** — they report `defaultSource:"class_cdo"` and `defaultValue` equal to whatever the CDO currently holds. So after any `property.set` mutates the CDO, the reported `defaultValue` tracks the *mutated* value, and `property.reset` copies that value onto itself — a guaranteed no-op. It returns success with `wasOverridden:false`, `oldValue == defaultValue`, `isOverridden:false`, having restored nothing. The pre-edit value is unrecoverable through the MCP even though the engine still holds it on the parent-class CDO.

This is the same defect class as the DONE ticket `B-property-component-parent-default` (which fixed inherited Blueprint **component templates** to resolve the nearest **parent component template** with `defaultSource:"parent_template"`), but for the **CDO root object** itself the resolver still falls back to `class_cdo` (self) instead of the parent-class CDO / archetype. The component-template fix did not cover the top-level CDO case.

Note: `property.set` recompiling+baking the value into the CDO is documented and correct (`property.set` wiki: "Recompiles the Blueprint when the target is a CDO."). The bug is on the **reset/default-resolution** side: a CDO is its own class default only for *locally authored* properties; for an *inherited, locally-unmodified* property the default must come from the archetype, not from self.

## Impact

The documented tweak-and-revert round-trip (`property.set` then `property.reset`) cannot restore a Blueprint CDO scalar that was inherited from a parent class back to its factory value. `property.reset` silently succeeds with no effect, and `defaultValue` in `property.list`/`property.get` misreports the post-edit value as the default — so a caller cannot even detect that the field is now off its inherited default.

## Repro (verbatim, replay-confirmed against mcp__editor-automation__call)

Target: `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic.BP_Light_Bulb_Basic` (resolves to `Default__BP_Light_Bulb_Basic_C`). `ParentClass` of `BP_Light_Bulb_Basic_C` is `/Script/Engine.Actor` (verified via `system.inspect.inspect_object`), so `SpriteScale` is inherited from `AActor`, whose CDO default is `1`. The bulb's own CDO does NOT author a local `SpriteScale` override.

1. `property.set { objectPath: "...BP_Light_Bulb_Basic", propertyName: "SpriteScale", value: 7 }`
   → `{ saved: true, value: 7 }` (recompiles the CDO — documented/expected).
2. `property.get { ...SpriteScale, includeDefault: true, includeOverrideState: true }`
   → `{ value: 7, defaultSource: "class_cdo", defaultValue: 7, isOverridden: false }`
   — the CDO's own (mutated) value is reported AS the default; `isOverridden` stays false.
3. `property.reset { ...SpriteScale }`
   → `{ oldValue: 7, defaultValue: 7, defaultSource: "class_cdo", wasOverridden: false, isOverridden: false }`
   — no-op: it copied the CDO's value onto itself. The archetype value (`AActor` CDO `SpriteScale = 1`) was never consulted.
4. `property.get { ...SpriteScale, includeDefault: true, includeOverrideState: true }`
   → `{ value: 7, defaultSource: "class_cdo", defaultValue: 7, isOverridden: false }`
   — still 7; the editor reset-arrow would have restored 1.

(The same starting state reproduces from any prior `property.set`; an earlier fuzz iteration had already baked `SpriteScale = 4` into this CDO, confirming the mutation persists in-session and the reported `defaultValue` tracks it.)

## Expected

For a Blueprint generated-class CDO, when a property has no locally authored value (it equals — or, pre-edit, equalled — the value on the property's archetype / parent-class CDO), `property.reset` should restore the **archetype** value (mirroring the editor reset arrow and `UObject::GetArchetype()`), and `property.list` / `property.get` should report `defaultSource` distinguishing the archetype/parent-class default from a true self default. `property.reset` should report `wasOverridden: true` (and a differing `oldValue`/`defaultValue`) when the CDO value differs from its archetype.

## Fix (proposed)

In the CDO branch of the default-resolution path (`UtilityPropertyHandler.cpp` / `PropertyUtils.cpp`, the same site that grew the `parent_template` source for `B-property-component-parent-default`): when the resolved object is a `UBlueprintGeneratedClass` CDO, resolve the per-property default from `CDO->GetArchetype()` (the parent-class CDO) rather than from the CDO itself. Emit a distinct `defaultSource` (e.g. `archetype` / `parent_class_cdo`) so callers know reset targets the inherited default. Fall back to `class_cdo` (self) only when there is no meaningful archetype (e.g. the property is locally authored on this CDO, or the archetype value is identical).

## History
- `#2-cdo-archetype-default-source` `IN-REVIEW` developer — Fixed the default-resolution path so a Blueprint generated-class CDO root resolves the per-property default from its archetype (the parent-class CDO) instead of from itself. Added `ResolveBlueprintCdoArchetype()` (CDO with `RF_ClassDefaultObject` whose class is a `UBlueprintGeneratedClass` → `UObject::GetArchetype()` = `GetArchetypeForCDO()` = parent-class CDO, only when distinct from the CDO) and wired a new branch into `ResolveDefaultSourceObject` emitting `defaultSource:"parent_class_cdo"`. To avoid regressing Blueprint-local CDO variables (which do not exist on the parent class), `FResolvedDefaultSource` now carries a `FallbackObject`/`FallbackSource` (the self CDO / `class_cdo`); both `ResolveDefaultPropertyFromSource` overloads (used by property.get/reset and property.list, incl. the nested-path case) fall back to it when the property is absent on the archetype. Now `property.get`/`list` report `defaultSource:"parent_class_cdo"`, `defaultValue` = the inherited parent default, and `isOverridden` true when the CDO value differs; `property.reset` copies the parent default onto the CDO (`wasOverridden:true`) instead of self-onto-self. Non-CDO and component-template (`parent_template`) paths are unchanged (FallbackObject defaults null). File: `Source/EditorAutomationRpcGateway/Private/Handlers/Utility/UtilityPropertyHandler.cpp`. Test: `Source/EditorAutomationRpcGateway/Private/Tests/Utility/TestPropertyCdoParentClassDefault.cpp` (`FPropertyCdoParentClassDefaultTest`) builds a parent BP (AActor subclass) with an int member authored to 3 on the parent CDO, a child BP subclassing it, mutates the inherited value to 9 on the child CDO, then drives production property.get/reset/list and asserts source=`parent_class_cdo`, default=3, isOverridden flips, and reset restores 3 — it fails (back to `class_cdo`/no-op) if the fix is reverted.
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed via mcp__editor-automation__call on `BP_Light_Bulb_Basic` (`ParentClass=/Script/Engine.Actor`, so `SpriteScale` is an inherited `AActor` field, archetype default 1). `property.set SpriteScale=7` recompiles the CDO (documented), then `property.get` reports `defaultSource:"class_cdo", defaultValue:7, isOverridden:false`, and `property.reset` returns `oldValue:7, defaultValue:7, wasOverridden:false, isOverridden:false` — a no-op that never consults the `AActor` archetype (`SpriteScale=1`). Final `property.get` still reads `value:7`. The reset resolves the default from the CDO's own (mutated) value instead of the parent-class archetype, so a CDO scalar inherited from a parent class can never be reset to its factory default and `defaultValue` misreports the post-edit value as the default. Distinct from DONE `B-property-component-parent-default` (that covered inherited *component templates* → parent *component template*; this is the top-level CDO root → parent-class CDO / archetype, an uncovered branch of the same resolver).
