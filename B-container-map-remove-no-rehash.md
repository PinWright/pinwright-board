---
id: B-container-map-remove-no-rehash
title: "container.map read scans (get / get_keys / has_key / remove) iterate to Helper.Num() not Helper.GetMaxIndex() — after a remove leaves a gap, a survivor at a high internal index is silently skipped and 'vanishes'"
status: IN-REVIEW
severity: High
category: bug
tags: [container, map, set, getmaxindex, sparse, corruption, silent]
---

# `container.map` (and `container.set`) read/scan loops bound iteration by `Helper.Num()` instead of `Helper.GetMaxIndex()` — a removed key leaves a sparse-array gap, so a surviving entry living at internal index ≥ `Num()` is never visited and appears to have vanished

The four `container.map` scan loops walk raw internal indices with
`for (int32 i = 0; i < Helper.Num(); ++i)` + `if (!Helper.IsValidIndex(i)) continue;`
— `get` (`UtilityPropertyHandler.cpp:2306`), `remove` match scan (`:2374`),
`has_key` (`:2428`), `get_keys` (`:2475`). The `IsValidIndex` guard shows the
author knew the storage can contain holes, **but the loop *bound* is wrong**:
a `FScriptMap`/sparse set has **gaps** in its internal index space after a
`RemoveAt`, and the canonical bound for a gapped scan is `Helper.GetMaxIndex()`
(documented `≥ Num()`, `UnrealType.h:4761-4774`, `checkSlow(Result >= Num())`),
not `Helper.Num()` (the live-element *count*).

`FScriptMapHelper::RemoveAt` (`UnrealType.h:5090-5135`) on this UE 5.7 tree takes
the **non-compact `#else` branch** — `UE_USE_COMPACT_SET_AS_DEFAULT` is **0**
(`ContainerAllocationPolicies.h:1647-1648`) — which calls
`Map->RemoveAt(LocalIndex, MapLayout)` (`:5124-5132`). That path **does not**
swap the last element into the freed slot; it frees the slot in place, leaving
an invalid hole at that internal index while surviving elements keep their
original internal indices. The very next engine comment confirms it
(`:5143-5144`): *"Maps have gaps in their indices…"*. The engine's own
`FindInternalIndex` (`:5148-5177`) branches on exactly this — *"if map is compact,
use random access"* `if (Num() == GetMaxIndex())` — and otherwise iterates to
`Map->GetMaxIndex()` skipping `!IsValidIndex` slots. After a remove that leaves
a gap, `Num() < GetMaxIndex()`, so `for i < Num()` under-iterates by exactly one
index and **silently drops the highest-indexed survivor**.

This is a pure linear-scan iteration bug: the data and the hash are fully intact
(the engine maintains the hash inside `RemoveAt`; no `Rehash()` is needed or
relevant here). `mapSize` (`Helper.Num()`, the element count) is reported
*correctly*; it is the `get_keys`/`has_key`/`get` scans that under-report
because they stop one index short. Every call returns `success`/`isError:false`,
so the corruption is invisible: the tool says the map is fine while a single
untouched entry has become unreadable.

The `container.set` read scans share the identical defect: `container.set.remove`
(`:2636`) and `container.set.contains` (`:2723`) also use
`for i < Helper.Num()` + `IsValidIndex`. A `set.remove` that leaves a gap makes
a high-indexed survivor invisible to `contains`/the next `remove` the same way.
(`set.add` is unaffected — it uses `Helper.AddElement`, which hashes/dedupes
internally and does not hand-scan.) The previously WONTFIX'd
`B-container-set-add-remove-no-rehash` correctly refuted the *no-Rehash hash-desync*
theory but stopped there; its specific repro (3 elements, remove the middle one)
happened not to push a survivor past the `Num()` boundary, so the latent
loop-bound bug went unnoticed. This ticket fixes the bound in both namespaces.

## Repro (verbatim, replayed live against the running editor)

Target: a Blueprint CDO `TMap<FString,int32>`. Created
`/Game/Config/BP_ResourceCostsReplay` (parent Actor) with a
`map<string,int>` variable `ResourceCosts`, compiled; CDO path is
`/Game/Config/BP_ResourceCostsReplay.Default__BP_ResourceCostsReplay_C`.

```
# build 5 entries — internal indices Wood=0, Stone=1, Iron=2, Gold=3, Crystal=4
container.map.set {... key:"Wood"  value:5}   -> {mapSize:1}
container.map.set {... key:"Stone" value:10}  -> {mapSize:2}
container.map.set {... key:"Iron"  value:25}  -> {mapSize:3}
container.map.set {... key:"Gold"  value:100} -> {mapSize:4}
container.map.set {... key:"Wood"  value:8}   -> {mapSize:4}   # upsert, dedupe OK
container.map.set {... key:"Crystal" value:250} -> {mapSize:5}
container.map.get_keys {...} -> {keys:["Wood","Stone","Iron","Gold","Crystal"], keyCount:5}   # all 5 present

# remove ONE key — "Gold" (internal index 3); "Crystal" is the last element at internal index 4
container.map.remove {... key:"Gold"} -> {mapSize:4}   # Num()==4, correct — only Gold removed

# CORRUPTION — after removal, internal index 3 is a GAP and Crystal stays at index 4.
# get_keys loops `for i<Num()` == `for i<4`, visiting indices 0,1,2,3 (3 is the skipped
# gap) and NEVER index 4 — so "Crystal" is omitted even though it is still live:
container.map.get_keys {...} -> {keys:["Wood","Stone","Iron"], keyCount:3}   # WRONG: 3, not 4; "Crystal" skipped
container.map.has_key {... key:"Crystal"} -> {hasKey:false}                  # WRONG: was true before the remove
container.map.has_key {... key:"Gold"}    -> {hasKey:false}                  # (correct — was removed)
container.map.get     {... key:"Crystal"} -> [KEY_NOT_FOUND] Key 'Crystal' not found in map.
```

The state is self-contradictory only because the scans under-iterate:
`remove` reports `mapSize:4` (`Num()` — correct), while `get_keys`/`has_key`/`get`
report 3 keys (the scan stopped at `i < 4`, never reaching the survivor at
internal index 4). The map's storage and hash are intact; re-`set`ting Crystal
would re-add it at a new index but the read scans still under-iterate whenever
`Num() < GetMaxIndex()`.

## Impact

Silent data corruption on the read paths of the `container.map` (and
`container.set`) namespaces whenever a prior `remove` has left a gap. Because
every call succeeds (`isError:false`), an agent editing a map incrementally
(the documented designer-iteration workflow: set a few entries, remove one) has
no signal that a read-back is missing an untouched entry. `get_keys`/`get`/`has_key`
— exactly what a caller reads back to confirm — are the views that drop the
highest-indexed survivor.

## Fix

Change the scan bound from `Helper.Num()` (the live-element count) to
`Helper.GetMaxIndex()` (the non-inclusive max internal index, documented
`≥ Num()`) in every hand-rolled `for i + IsValidIndex` scan over a map/set's
internal index space:
- `container.map.get`   (`UtilityPropertyHandler.cpp:2306`)
- `container.map.remove` match scan (`:2374`)
- `container.map.has_key` (`:2428`)
- `container.map.get_keys` (`:2475`)
- `container.set.remove` (`:2636`)
- `container.set.contains` (`:2723`)

The existing `if (!Helper.IsValidIndex(i)) continue;` guard is already correct
and must stay — it skips the gap slots that `GetMaxIndex()` now exposes. No
`Rehash()` is involved: the engine maintains the hash inside `RemoveAt`, and
these scans are linear (not hash-backed), so the bug and its fix are purely
about the iteration bound. (`container.set.add` uses `Helper.AddElement` and is
unaffected; `map.clear`/`set.clear` use `EmptyValues` and are unaffected.)

Add a regression test driving a reflected `TMap<FString,int32>` (and a
`TSet<FName>`) through the production handlers: insert N entries, remove a key
positioned so the survivor lands at internal index ≥ `Num()` (i.e. remove a
*non-last* key after building enough entries), then assert `get_keys`/`has_key`
(map) and `contains` (set) still see the survivor. With the bound left at
`Helper.Num()` the survivor is skipped and the test fails.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live against a Blueprint-CDO
  `TMap<FString,int32>` (`/Game/Config/BP_ResourceCostsReplay`, `map<string,int>`
  var `ResourceCosts`, compiled). Built {Wood,Stone,Iron,Gold,Crystal} (5 keys,
  `get_keys` confirmed 5); `remove("Gold")` reported `mapSize:4` but `get_keys`
  then returned only `["Wood","Stone","Iron"]` (keyCount:3) — the untouched key
  **"Crystal" silently vanished**; `has_key("Crystal")`→false (was true),
  `get("Crystal")`→`KEY_NOT_FOUND`. (Reporter's stated root cause — a missing
  `Helper.Rehash()` / compact-set swap path — was incorrect; see #2.)
- `#2-reword-loop-bound` `OPEN` developer — Reworded: the reporter's repro is a
  **real, deterministic defect**, but its stated mechanism (missing `Rehash()`,
  compact-set swap leaving a stale hash, `AddPair` precondition) is **wrong** and
  refuted against UE 5.7 source. `UE_USE_COMPACT_SET_AS_DEFAULT` is `0`
  (`ContainerAllocationPolicies.h:1648`), so `FScriptMapHelper::RemoveAt`
  (`UnrealType.h:5090`) takes the non-compact `#else` branch (`:5124-5132`) which
  frees the slot **in place**, leaving a gap (engine comment `:5143`: *"Maps have
  gaps in their indices"*) and keeping the hash consistent — no `Rehash()` is
  needed. The actual root cause: the scan loops bound iteration by `Helper.Num()`
  (element count) when a gapped map needs `Helper.GetMaxIndex()` (`:4761`,
  `checkSlow(Result >= Num())`); the engine's own `FindInternalIndex` (`:5148`)
  iterates to `GetMaxIndex()` for non-compact maps. After removing a non-last key
  the highest-indexed survivor lives at internal index `== Num()` and is skipped
  by `for i < Num()` — exactly the repro. Title/body/Fix/tags reworded
  accordingly (Rehash theory removed; `getmaxindex`/`sparse` tags added).
- `#3-fix-getmaxindex-bound` `IN-REVIEW` developer — Fixed: changed the scan
  bound from `Helper.Num()` to `Helper.GetMaxIndex()` in all six hand-rolled
  `for i + IsValidIndex` internal-index scans —
  `container.map.get` / `.remove` / `.has_key` / `.get_keys` and
  `container.set.remove` / `.contains` (`UtilityPropertyHandler.cpp`). The
  `IsValidIndex(i)` skip guards were already present and are retained; no other
  change (no `Rehash()`, `set.add`/`*.clear` untouched). Added regression test
  `Tests/Utility/TestContainerMapGapIteration.cpp` (+ fixture
  `TestContainerMapGapHost.h`): drives the production `container.map.set/remove/
  get_keys/has_key/get` handlers on a reflected `TMap<FString,int32>` and the
  production `container.set.add/remove/contains` handlers on a `TSet<FName>`,
  building enough entries that the removed key is non-last so a survivor sits at
  internal index ≥ `Num()`, then asserts the survivor is still visible via
  `get_keys`/`has_key`/`get`/`contains` and that `Num()` matches the visible
  count. Reverting the bound to `Helper.Num()` makes the survivor invisible and
  fails the test. Not compiled/run here (later phase verifies).
