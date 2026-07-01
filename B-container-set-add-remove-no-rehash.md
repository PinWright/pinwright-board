---
id: B-container-set-add-remove-no-rehash
title: "container.set.add/remove don't Rehash() — sparse-array count, contains, and property.get silently desync (set corruption)"
status: WONTFIX
severity: High
category: bug
tags: [container, set, rehash, corruption, fname, property]
---

# container.set.add / container.set.remove never call FScriptSetHelper::Rehash() — the set's hash index desyncs from its sparse array, silently corrupting the set

`container.set.add` mutates a `TSet` UPROPERTY via
`FScriptSetHelper::AddElement(TempElem)` with **no following `Helper.Rehash()`**
(`UtilityPropertyHandler.cpp` ~line 2590), and `container.set.remove` mutates
via `Helper.RemoveAt(i)` with no rehash either (~line 2676). `AddElement`
appends into the sparse array but does **not** update the set's hash index, and
`RemoveAt` leaves the hash index referring to compacted/removed slots. After a
mix of adds and removes, the three views of the same set diverge:

- **`setSize`** (`Helper.Num()`, the sparse-array element count) reports one
  count,
- **`container.set.contains` / `container.set.remove`** (a *linear scan* over
  `Helper.GetElementPtr(i)` for `i < Helper.Num()`) reports membership off the
  raw sparse array,
- **`property.get`** (hash-backed `ExportText`, the source of truth a caller
  reads back) reports a *different* set — it collapses to the elements the stale
  hash can still reach, **silently dropping members**.

Every call returns `success` / `isError:false`, so the corruption is invisible:
a `tools/call` reports the set was updated while the persisted set is wrong.
This makes the documented "per-element set lifecycle" (`add` repeatedly, then
`remove` one entry) unusable — the set ends up holding the wrong elements with
no error signal.

NOTE this is a **distinct defect** from `B-container-set-fname-lookup-broken`
(IN-REVIEW), which is about the contains/remove element-match loops lacking an
`FNameProperty` branch (already fixed in this build — FName matching works). That
ticket explicitly asserts *"`container.set.add` correctly stores elements …
dedupe is a correct no-op"*; this ticket shows that assertion is wrong once a
`remove` is interleaved: the missing `Rehash()` corrupts the set regardless of
the lookup branch. They are independent: fixing the lookup branch does not fix
the hash desync.

## Repro (verbatim, replayed live against the running editor)

Target is a real engine-native reflected `TSet<FName>`
(`/Script/Engine.Default__AssetManagerSettings.MetaDataTagsForAssetRegistry`),
present in every project — NOT a Blueprint CDO, so this is not BP-specific.

```
# clean start
container.set.clear   {objectPath:"/Script/Engine.Default__AssetManagerSettings", propertyName:"MetaDataTagsForAssetRegistry"}
   -> {previousSize:0, newSize:0}

# add three FName elements — setSize grows 1/2/3, property.get agrees (still consistent here)
container.set.add  {... value:"Forest"}  -> {setSize:1}
container.set.add  {... value:"Cavern"}  -> {setSize:2}
container.set.add  {... value:"Summit"}  -> {setSize:3}
property.get       {...}                  -> {value:["Forest","Cavern","Summit"]}   # OK so far
container.set.contains {... value:"Summit"} -> {contains:true}    # OK
container.set.contains {... value:"Desert"} -> {contains:false}   # OK

# now interleave a remove + an idempotent re-add — this triggers the desync
container.set.remove {... value:"Cavern"} -> {setSize:2}          # remove OK by count
container.set.add    {... value:"Summit"} -> {setSize:2}          # idempotent no-op by count

# CORRUPTION — the three views now disagree:
property.get            {...}              -> {value:["Forest"]}   # WRONG: collapsed to 1, lost "Summit"
container.set.contains  {... value:"Forest"} -> {contains:true}
container.set.contains  {... value:"Summit"} -> {contains:false}  # WRONG: setSize:2 + just re-added, but linear scan can't find it
container.set.contains  {... value:"Cavern"} -> {contains:false}  # (correct — was removed)
```

Final state is self-contradictory: `setSize:2`, `property.get`→`["Forest"]`
(size 1), `contains("Summit")`→false even though it was the element re-added.
The sparse array, the hash-backed export, and the count are all out of sync.

## Impact

Silent data corruption on the core write path of the `container.set` namespace.
Because every call succeeds, an agent curating a set incrementally (the exact
documented workflow) has no way to detect that the set it persisted is wrong.
`property.get` (what a caller reads back to confirm) is the view that loses
elements, so the user sees a confidently-wrong final set.

## Fix

Make the mutating handlers leave the hash index consistent with the sparse
array on every exit:

- `container.set.add`: after `Helper.AddElement(TempElem)` call
  `Helper.Rehash()` — or replace the raw `AddElement` with
  `Helper.FindOrAddElement(...)` (which hashes and dedupes correctly), and
  rehash if needed. The current raw `AddElement` is documented engine-internal
  usage that REQUIRES a subsequent `Rehash()`.
- `container.set.remove`: after `Helper.RemoveAt(i)` call `Helper.Rehash()` so
  later `add`/`contains`/`property.get` see a consistent index.

Add a regression test driving a reflected `TSet<FName>` through
add×3 → remove(one) → add(existing) and asserting all three views agree:
`Helper.Num()`, `container.set.contains` for each expected member, and
`property.get` / `ExportText` return the SAME element set
(`{Forest, Summit}` after the repro above). Reverting the `Rehash()` calls
must fail the test.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live against
  `/Script/Engine.Default__AssetManagerSettings.MetaDataTagsForAssetRegistry`
  (`TSet<FName>`, engine-native, not a BP CDO). add×3 → property.get all agree
  (`[Forest,Cavern,Summit]`); after `remove("Cavern")` (setSize:2) +
  idempotent `add("Summit")` (setSize:2) the views desync:
  `property.get`→`["Forest"]` (lost Summit), `contains("Summit")`→false,
  `setSize`:2. Root cause confirmed in source: `container.set.add` uses
  `FScriptSetHelper::AddElement` and `container.set.remove` uses `RemoveAt`,
  neither followed by `Helper.Rehash()`, so the hash index desyncs from the
  sparse array and the hash-backed `ExportText`/`property.get` silently drops
  members. Distinct from `B-container-set-fname-lookup-broken` (the FName
  lookup-branch fix is already present in this build and the FName branches
  match correctly). All calls return success — silent corruption.
- `#2-not-a-bug-engine-self-rehashes` `WONTFIX` developer — Not a defect:
  all three load-bearing mechanisms are refuted against UE 5.7 engine source
  and this plugin's read path. (1) `FScriptSetHelper::AddElement`
  (`UnrealType.h:6126`) does NOT raw-append — it calls `Set->Add(...)` with
  hash + identity functors → `TScriptSparseSet::Add` → `AddNewElement`
  (`ScriptSparseSet.h:316-340`), which links the new element into the hash
  bucket chain (`:331-336`) or rehashes when buckets must grow (`:323-328`);
  the hash index is always consistent after `AddElement`. The
  `_NeedsRehash` caveat the ticket invokes belongs to
  `AddDefaultValue_Invalid_NeedsRehash`/`AddUninitializedValue`
  (`UnrealType.h:5903-5917`), which the handler never calls. (2)
  `FScriptSetHelper::RemoveAt` (`UnrealType.h:5932`) runs the non-compact
  `#else` path (`UE_USE_COMPACT_SET_AS_DEFAULT==0`,
  `ContainerAllocationPolicies.h:1648`) → `TScriptSparseSet::RemoveAt`
  (`ScriptSparseSet.h:147`), whose FIRST action unlinks the element from its
  hash bucket chain (`:153-161`) BEFORE removing it from the elements array
  (`:164`) — the hash stays consistent. No `Helper.Rehash()` is needed after
  either `AddElement` (`UtilityPropertyHandler.cpp:2590`) or `RemoveAt`
  (`:2676`). (3) The "three diverging views" symptom is impossible because
  none of them is hash-backed: `property.get` for a set
  (`Utils/PropertyExport.cpp:1022-1032`) iterates the sparse array directly
  (`for i<Helper.Num()` + `Helper.GetElementPtr(i)`) — the SAME iteration
  `container.set.contains` (`:2723-2759`) and `container.set.remove`
  (`:2636-2688`) use; all three read identical data and cannot disagree. The
  repro's `contains("Summit")→false` is the already-fixed
  `B-container-set-fname-lookup-broken` symptom (the FName branch is present at
  `UtilityPropertyHandler.cpp:2665-2672` and `:2751-2758`), not a hash desync.
  The existing regression `Tests/Utility/TestContainerSetFNameLookup.cpp:96-99`
  drives the production `container.set.remove` handler then asserts a genuine
  hash-backed `TSet::Contains(...)==false` and `Num()==2` — which would FAIL if
  `RemoveAt` left the hash stale; it passes, empirically proving the hash is
  maintained. The proposed `Rehash()` fix patches a non-bug (engine-private
  cost for zero payoff) and the proposed test could not fail against current
  code, so it could not guard the claimed behavior. No code change.
