---
id: F-property-list-name-filter
title: "property.list has no name-filter parameter; full CDO listing is verbose"
status: DONE
severity: Low
category: feature
tags: [property, ergonomic, filter]
---

# `property.list` needs a name-filter parameter

`property.list` returns every UPROPERTY on the target object. On
Blueprint CDOs with many fields (e.g. `B_DroneGameInstance` has 55
properties), the response is large and forces the caller to grep
the result for the names of interest. There's no built-in way to
narrow the output by name pattern.

The currently accepted parameters are `objectPath`, `includeAll`,
`editableOnly`, `includeReadOnly`, `includeTransient`,
`includeValues`, `includeDefault`, `includeOverrideState`,
`includeMetadata`, `omitOversized`. None of them filter by name.

Trying `{filter: "Forced"}` or `{nameMatch: "Forced"}` returns:

```
UNKNOWN_PARAMS: Unknown parameter(s) for 'property.list': [filter].
Valid parameters: [objectPath, includeAll, editableOnly, ...]
```

## Why this matters

- Auditing one field family (e.g. all `Forced*` or `bIs*` flags) on
  a CDO with 50+ properties returns 50+ entries instead of the 4 of
  interest. Wastes tokens and forces post-filtering.
- Common workflow: caller knows the field name (or a stem) and wants
  to verify current value, override state, and metadata. With no
  filter, the response is mostly noise.

## Proposed

Add `nameMatch` (case-insensitive substring) and/or `nameRegex`
(full regex) parameters to `property.list`. Filter applies to the
UPROPERTY `FName`, not the friendly display name. Both forms can
coexist; substring is the simple case, regex covers wildcard /
prefix-or-suffix needs.

```
{
  objectPath: "/App/.../B_DroneGameInstance",
  nameMatch: "Forced",
  includeValues: true
}
```

Returns only properties whose name contains `Forced`
(case-insensitive). Same response shape as today, just shorter.

If `nameMatch` and `nameRegex` are both provided, regex wins (or
error on ambiguity — either is fine).

Test: a transient UObject with 20+ properties, half matching a stem.
Assert filter returns the right subset.

## Related

- `widget.describe` already has structured filters for its widget
  tree traversal — same general pattern, different domain.
- For batch inspection without filters, `property.get` already supports
  one-property reads. The filter is the missing middle ground between
  "one property" and "every property."

## History
- `#1-initial-feature-request` `OPEN` reporter — Session needed to verify the new `Forced*` UPROPERTYs on `DA_Chemical_GasLeak` (a `ULyraUserFacingExperienceDefinition` with 34 properties) and on `B_DroneGameInstance` CDO (55 properties). Both calls returned full lists; relevant entries had to be located by scrolling/grepping the response. A `nameMatch: "Forced"` parameter would have returned 4 entries instead of 34/55. Tried `{filter: "Forced"}` first and got `UNKNOWN_PARAMS` — confirmed no filter parameter exists in the current handler.
- `#2-additional-componenttags` `OPEN` reporter — Additional evidence: session needed only `ComponentTags` on `/App/HELIOS/Drones/Atlas/B_Geoscan801.B_Geoscan801_C:Motors: FR Move_GEN_VARIABLE`, but `property.list` rejected `propertyNames:["ComponentTags"]` with `UNKNOWN_PARAMS` and returned the full 202-property component listing. A single-name or name-match filter would have reduced the response to the relevant field and avoided manual scanning.
- `#3-name-filter-params` `IN-REVIEW` developer — Added 'nameMatch' (case-insensitive substring) and 'propertyNames' (exact-name allow-list) optional params to property.list in UtilityPropertyHandler.cpp. AND semantics when both supplied; empty = no-op. Skipped 'nameRegex' (YAGNI). Test fixture UTestPropertyListNameFilterHost + Tests/Utility/TestPropertyListNameFilter.cpp covers substring, case-insensitivity, exact-list, AND combo, and no-filter baseline.
- `#4-verify-fix` `DONE` tester — Verified via live property.list against /App/App/LevelBlueprints/B_DroneGameInstance.B_DroneGameInstance_C: schema (property.list?) exposes both new params; nameMatch="cooked" returned 4 properties (incl. bHasCookedComponentInstancingData and CachedCookedMetaDataPtr, confirming case-insensitive substring); propertyNames=["Timelines","UberGraphFunction","NotARealProperty"] returned exactly the 2 existing names; AND combo returned the single intersecting name CookedPropertyGuids.
