---
id: E-set-default-rejects-container-typed-properties
title: "blueprint.set_default cannot assign a map-typed (and presumably any container-typed) property: CONVERSION_FAILED 'Unsupported property type for JSON assignment', with no list of what IS supported"
status: DONE
severity: Medium
category: ergonomic
tags: [blueprint, set-default, container, map, tmap, json, conversion-failed, unactionable-error, workaround-python]
encounters: 2
costly: 2
lastSeen: 2026-09-05T20:15:00Z
---

# The verb takes JSON but refuses the types JSON is best at

`blueprint.set_default` documents `value` as "JSON-compatible string; type-coerced to the property's
type via reflection". Handed a JSON object for a `TMap` property, it fails:

```
blueprint.set_default {
  path: "/Game/FPS/Weapons/BP_WeaponBase",
  propertyName: "ImpactDecalSizes",
  value: "{\"SurfaceType1\": 14.0, \"SurfaceType2\": 14.0, \"SurfaceType3\": 14.0,
           \"SurfaceType4\": 14.0, \"SurfaceType6\": 30.0, \"SurfaceType7\": 10.0}"
}
-> [CONVERSION_FAILED] Unsupported property type for JSON assignment
```

The property is `map<enum<EPhysicalSurface>, float>`, created moments earlier by
`blueprint.add_variable` — which **accepted that exact type token without complaint**. So one verb
in the pair creates container variables and the other cannot populate them.

## Two things make the error unactionable

**It does not say what IS supported.** Compare `blueprint.add_variable`, which on a bad type token
answers with the full accepted grammar — primitives, builtin structs, short names, full paths, and
the `array<T>` / `set<T>` / `map<K,V>` wrappers. That error teaches; this one does not. A caller
cannot tell from it whether the fault is the container, the enum key, the float value, the JSON
spelling, or the property itself.

**It does not say whether the property was touched.** The response carries no `changed` or
`previousValue`, so after a `CONVERSION_FAILED` the caller does not know if a partial write landed.
`B-container-array-conversion-failure-leaves-element` records exactly that failure mode for arrays
on a sibling verb, which is reason to state it here rather than assume it is clean.

## Scope not established

Only `TMap` was hit. Whether `TArray` and `TSet` fail the same way through this verb was not
tested — the workaround was found first and the build moved on. Worth one probe each before fixing,
since the fix is likely one shared code path.

## Workaround

Set the container through `python.execute` on the CDO, which works and is what this build shipped:

```python
cdo = unreal.get_default_object(bp.generated_class())
cdo.modify(True)
cdo.set_editor_property('ImpactDecalSizes', {unreal.PhysicalSurface.SURFACE_TYPE1: 14.0, ...})
unreal.PinWrightPackageLibrary.mark_package_dirty(bp)
unreal.EditorAssetLibrary.save_asset(path, only_if_is_dirty=False)
```

That costs the caller the reinstancing guard `blueprint.set_default` now carries
(`allowReinstancing`, which refuses when another stream holds live instances). **So the workaround
is strictly less safe than the verb**, which is the real argument for fixing this rather than
documenting it: an agent driven to `python.execute` for containers loses a protection that exists
precisely because a compile in that situation once killed a shared editor.

## Fix

Accept JSON objects for `TMap` and JSON arrays for `TArray`/`TSet`, keyed and valued through the
same reflection coercion the scalar path already uses. Failing that, name the supported set in the
error the way `add_variable` does, and say whether the property was modified.

## History
- `#1-map-typed-property-rejected` `OPEN` reporter - Found on the FPS build, 2026-09-05, adding a per-surface decal-size map to `BP_WeaponBase`. `blueprint.add_variable` accepted `map<enum<EPhysicalSurface>,float>` and created the variable; `blueprint.set_default` then refused a JSON object for it with `[CONVERSION_FAILED] Unsupported property type for JSON assignment` and no indication of the supported set or whether anything was written. Worked around via `python.execute` on the CDO, which succeeded and was byte-verified on disk (six map entries: f32 14.0 x4, 30.0 x1, 10.0 x1). The workaround loses the verb's `allowReinstancing` guard, which had refused a compile earlier in the same session because another stream held three live instances in its PIE world — so being pushed off the verb costs a real safety net. `TArray`/`TSet` not tested. Distinct from `E-set-default-none-clear-conversion-failed` (clearing an object reference with "None") and from `B-container-array-conversion-failure-leaves-element` (partial write on a sibling verb), though the latter is why "was the property touched" is worth answering here.
- `#2-array-scope-and-a-safer-workaround` `OPEN` reporter — **Closes the ticket's "Scope not established" question for `TArray`, and it fails for a different reason than the map does.** Setting `Tags` (`TArray<FName>`) on `/Game/FPS/Weapons/Test/BP_WeaponTestPawn` was refused three times, once per plausible encoding:

  ```
  value: "[\"FPS_Team0\"]"     -> [CONVERSION_FAILED] Expected array for array property
  value: "(\"FPS_Team0\")"     -> [CONVERSION_FAILED] Expected array for array property   # UE ImportText form
  value: "FPS_Team0"           -> [CONVERSION_FAILED] Expected array for array property
  ```

  The map path says *"Unsupported property type for JSON assignment"*; the array path says *"Expected array for array property"* — i.e. the array branch **is** implemented and is asking for a JSON array, but the schema types `value` as `string`, so **no value a caller can send will ever satisfy it**. The verb is asking for something its own parameter type forbids. That makes the array case arguably worse than the map case: the error is actionable-sounding and leads the caller through encodings that cannot work, rather than admitting the type is unreachable.

  **A workaround that keeps the reinstancing guard, which the `#1` python route loses.** `property.set` types `value` as `any` and its wiki states "For array properties, pass a JSON array; the call replaces the entire array". Pointed at the CDO it works, and the compile is then done by `blueprint.compile`, which carries the guard:

  ```
  property.set {objectPath: "/Game/FPS/Weapons/Test/BP_WeaponTestPawn.Default__BP_WeaponTestPawn_C",
                propertyName: "Tags", value: ["FPS_Team0"], markDirty: true}
    -> {"applied":true,"markedDirty":true,"value":["FPS_Team0"]}
  blueprint.compile {path: "/Game/FPS/Weapons/Test/BP_WeaponTestPawn"}
    -> {"compiled":true,"status":"UpToDate","errors":[]}          # guard still applies, and would have refused
  asset.save    {assetPath: "...", force: true}
    -> {"saved":true,"saveState":"written","sizeBytes":179910}
  ```

  Byte-verified on disk afterwards: `FPS_Team0` present in `BP_WeaponTestPawn.uasset`, mtime moved. **Recommend replacing `#1`'s `python.execute` workaround with this three-call one in the ticket body** — it is the same three steps `blueprint.set_default` performs internally, minus the broken coercion, and it gives up none of the safety.

  Note the `value` schema type is the actual bug surface for arrays: whatever the coercion fix is, `value` has to stop being `string` (or the handler has to parse a JSON string into an array before the type check) or the array branch stays unreachable.
- `#3-containers-accepted-validate-before-modify` `IN-REVIEW` developer — Both halves fixed, plus the "was it touched" question. (1) `blueprint.set_default` declares `value` as `any` (was `string`, which the dispatcher's declared-type gate refused for every JSON array/object, so the array branch was unreachable exactly as `#2` said); a string value aimed at an array/set/map property is decoded as JSON text first, so `"[\"FPS_Team0\"]"` now works too. (2) The shared importer `ApplyJsonValueToProperty` (`Utils/PropertyImport.cpp`) gained `TSet` (JSON array) and `TMap` (JSON object, keys by spelling — enumerator names, ints) branches; elements/keys/values go through the same reflection coercion, the container is built in a scratch value and copied over only when every entry converted, and a duplicate element/key is refused. This also extends `property.set`. The unsupported-type error now names the property class and the supported set. (3) `set_default` converts into a copy of the current value before `Modify()`; a failure is `CONVERSION_FAILED` ending `Property '<name>' (<cpp type>) was not modified.` and the Blueprint is untouched. Files: `Handlers/Blueprint/BlueprintPropertyHandler.cpp`, `Utils/PropertyImport.cpp`, new `Tests/Blueprint/TestBlueprintSetDefaultContainers.cpp`, `docs/wiki-src/blueprint.md` (`#### Container values`), `docs/wiki-src/property.md`, `CHANGELOG.md`. Tests: `PinWright.blueprint.set_default.ValueSlotAdmitsJsonContainers`, `PinWright.blueprint.set_default.MapArraySetDefaultsApplied` (the ticket's `map<enum<EPhysicalSurface>,float>`, `array<name>`, `set<int>` from JSON text), `PinWright.blueprint.set_default.BadMapKeyRefusedWithoutWriting`.
- `#4-review-fixes` `IN-REVIEW` developer — Review rounds folded in. (1) Round 1: the scratch pre-check is skipped for Instanced subobject brace text, whose import builds a subobject outered to the container (a scratch base there was a crash); contract note on `ApplyJsonValueToProperty` in `Utils/PropertyImport.h`. `property.set` (`Handlers/Utility/UtilityPropertyHandler.cpp`) now stages set/map values before `Modify()`, so a failed write no longer dirties the object. (2) Round 2: the skip keyed on the property flag alone, so every component-typed variable (`CPF_InstancedReference` via `DefaultToInstanced`) lost the "refused before anything is modified" guarantee for a bad path string. Both the importer and `blueprint.set_default` now use one exported predicate, `IsInstancedSubobjectText(Property, Value)` (`Utils/PropertyImport.h/.cpp`): Instanced flag AND a string value starting with `{`. Docs (param description, `docs/wiki-src/blueprint.md`, `CHANGELOG.md`) state the one exception: Instanced brace text is validated in place, all-or-nothing on the property, but the Blueprint is marked modified. New test `PinWright.blueprint.set_default.InstancedNavigationBraceTextApplied` (Widget Blueprint CDO `Navigation` from `{Down={Rule=Stop},_kind=/Script/UMG.WidgetNavigation}`; asserts a `UWidgetNavigation` outered to the CDO, `RF_DefaultSubObject`, `Down.Rule == Stop`).
- `#5-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.blueprint.set_default.ValueSlotAdmitsJsonContainers`, `PinWright.blueprint.set_default.MapArraySetDefaultsApplied`, `PinWright.blueprint.set_default.BadMapKeyRefusedWithoutWriting`, `PinWright.blueprint.set_default.InstancedNavigationBraceTextApplied`, plus `PinWright.property.set.InstancedTextReplacesImportedOnBlueprintCdo`. Acceptance met: `value` is declared `any`, so JSON arrays/objects clear the type gate (the #2 array-unreachable case); the ticket's `map<enum<EPhysicalSurface>,float>`, `array<name>` and `set<int>` apply from JSON text; a bad map key is refused with `CONVERSION_FAILED ... was not modified` and the Blueprint untouched, answering the was-it-touched question. Coverage limit: the Instanced brace-text exception marks the Blueprint modified, as #4 documents.
