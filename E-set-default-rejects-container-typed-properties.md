---
id: E-set-default-rejects-container-typed-properties
title: "blueprint.set_default cannot assign a map-typed (and presumably any container-typed) property: CONVERSION_FAILED 'Unsupported property type for JSON assignment', with no list of what IS supported"
status: OPEN
severity: Medium
category: ergonomic
tags: [blueprint, set-default, container, map, tmap, json, conversion-failed, unactionable-error, workaround-python]
encounters: 1
lastSeen: 2026-09-05T19:51:39Z
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
