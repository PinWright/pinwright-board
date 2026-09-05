---
id: B-container-array-conversion-failure-leaves-element
title: "container.array.append and insert leave a new default element in the target array when value conversion returns an error"
status: IN-REVIEW
severity: High
category: bug
tags: [container, array, rollback, partial-mutation, property-import]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# Array conversion errors leave a partial mutation

## What happens

`container.array.append` calls `RootObject->Modify()`, immediately grows the live array with
`Helper.AddValue()`, and only then converts the supplied value
(`Handlers/Utility/UtilityPropertyHandler.cpp:2256-2272`). If
`ApplyJsonValueToProperty` returns false, `:2273-2277` sends `UNSUPPORTED_TYPE` and returns without
removing the new element.

`container.array.insert` has the same ordering: `Modify()` at `:2391`, live
`Helper.InsertValues(Index, 1)` at `:2401`, conversion at `:2411`, and an error return at
`:2412-2416` with no rollback. Insert also shifts every later element before reporting failure.
Neither failure path sends the change notification, but the UObject's array has already changed;
on an asset/CDO, a later save can persist the unwanted default slot.

## Why it matters

An error receipt falsely implies no edit landed. Retrying appends or inserts another slot and can
silently damage ordered asset data. Severity is High: the path requires an unsupported value shape,
but the resulting partial write is concrete and survives the failed call.

## What should happen

Convert into initialized temporary element storage first and mutate the live array only after
conversion succeeds. `container.map.set` already uses this fix shape at `:2581-2638`: initialize
temporary key/value storage, convert, then call `Helper.AddPair` once. If temporary conversion is
not feasible, remove the exact added/inserted slot on every failure and verify the original array
shape was restored. Add counterfactual tests that force conversion failure and compare the array
before and after.

**Workaround:** Read the property after any append/insert error and undo or reload the asset before retrying.

## Fix

PARTLY TRUE. At source commit `b4329838`, `container.array.append` and
`container.array.insert` already stage values before mutation, and whole-array
`property.set` stages replacement through `ApplyJsonValueToArrayDirect`. No
`container.array.resize` verb is registered. This change stages a complete copy of
the existing element before `container.array.set` applies JSON, then copies it into
the live array only after conversion succeeds; failures clean up the scratch value and
return `UNSUPPORTED_TYPE` without `Modify()` or notification. Added
`PinWright.container.array.set.MalformedStructNoMutation` alongside the existing
append/insert scalar tests. Changed files: `UtilityPropertyHandler.cpp`,
`TestPropertyImportMalformedScalars.cpp`,
`TestPropertyImportMalformedScalarsHost.h`, and `docs/wiki-src/container.md`.
No live, build, or test run was performed by this update.

## Related

- Catalog: `partial-mutation-without-complete-rollback`, `partial-nonatomic-success`
- `B-container-array-append-struct-element-crash` — sibling fixed wrong-pointer crash; it does not
  cover rollback when the now-correct conversion call returns false.

## History
- `#1-error-leaves-array-slot` `OPEN` reporter — Source-read append and insert from handler entry
  through their error exits and confirmed both mutate before conversion with no rollback. No asset
  was modified during this scan.
- `#2-stage-array-element` `IN-REVIEW` developer — PARTLY TRUE: fixed the remaining
  `container.array.set` set-at-index partial-struct mutation by staging a complete existing
  element; append/insert and whole-array replacement were already fixed at `b4329838`, and no
  `container.array.resize` verb exists. Added `PinWright.container.array.set.MalformedStructNoMutation`
  plus the existing scalar tests; no live/build/test run performed.
