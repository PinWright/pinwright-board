---
id: F-asset-search-native-subclass
title: "`asset.search` can't find BP subclasses of a native C++ class"
status: DONE
severity: Medium
category: feature
tags: [asset-search, class-filter, parent-class]
encounters: 2
lastSeen: 2026-06-30T15:54:09Z
---

# `asset.search` can't find BP subclasses of a native C++ class

`asset.search` with `classPathFilter="/Script/<Module>.<NativeClass>"` returns 0 results, even when BP assets whose `ParentClass` is that native class exist and are loaded. The filter is only matching the BP asset's own class tag (`Blueprint`), not descending through `ParentClass`.

**Repro (observed this session):**

Project has `/App/App/LevelBlueprints/B_RaceTrack`, a `Blueprint` asset whose `ParentClass` is `ADroneRacingTrack` (native C++).

```
asset.search(query="*", classPathFilter="/Script/App.DroneRacingTrack")
→ {"assets": [], "count": 0}

asset.search(query="*", classPathFilter="/Script/App.DroneRacingTrack",
             classFilterMode="contains")
→ {"assets": [], "count": 0}
```

Workaround: name-based globbing:
```
asset.search(query="B_*Track*", classFilter="Blueprint")
→ returns B_RaceTrack along with ~30 other track BPs — caller must
   then manually filter by checking each one's ParentClass.
```

That's O(N) work on the caller side to answer a question the asset registry can natively answer.

**Impact:** Blocks "find all BPs subclassing native class X" — a very common need when configuring class-ref properties on BP CDOs (e.g., setting `ADroneRacingTrack::ReplayEditorWidget` required finding `B_RaceTrack` first). Forced the session to fall back on name-globbing, which fails silently if the BP is named outside the expected pattern.

**Proposal:** Add one of:
1. A new parameter `parentClassPath: string` to `asset.search_assets` / `asset.search` that pushes an `FARFilter::ClassPaths` entry with `bRecursiveClasses=true` so the asset registry recursively matches BPs whose `ParentClass` (or any ancestor) is the given class.
2. Or, extend `classPathFilter` to accept native-class paths by internally converting `/Script/Module.NativeClass` → recursive class-path filter. Less explicit but preserves API shape.

Option 1 is clearer. The asset registry already supports recursive filtering — `IAssetRegistry::GetAssets` with `FARFilter{ClassPaths=[...], bRecursiveClasses=true}` is the canonical query — so the plumbing is a thin handler addition.

**Companion:** `blueprint.list` (separate tool) probably has the same gap; verify during fix.

## History
- `#1-native-class-filter-zero` `OPEN` reporter — Needed to locate the BP subclass of `ADroneRacingTrack` to set its new `ReplayEditorWidget` default via `blueprint.set_default`. `asset.search(classPathFilter="/Script/App.DroneRacingTrack")` and variants returned 0. Had to fall back to name search `B_*Track*`, eyeball the list, and guess `B_RaceTrack` (which happened to be correct). Would have silently missed a BP named outside the `B_*Track*` pattern.
- `#2-added-parent-class-param` `IN-REVIEW` developer — `asset.search` now accepts optional `parentClassPath` param. When set, resolves via `ResolveUClass`, builds `FARFilter` with `bRecursiveClasses=true`, and pre-filters `AllAssets` via the registry's native recursive class filter (same pattern blueprint.list uses). `classPathFilter` and `parentClassPath` are mutually exclusive — both set returns `INVALID_ARGUMENT`. Changed `AssetManageHandler.cpp:1022-1162`. Covered by `FAssetSearchNativeSubclassTest` and `FAssetSearchMutuallyExclusiveFiltersTest` in `TestAssetHandlers.cpp`. `blueprint.list` verified to already work correctly for this use case and is out of scope.
- `#3-returned-bp-asset-missed` `OPEN` tester — Returned: the parameter is accepted and mutual exclusion works, but the core ticket use case ("find the B_RaceTrack BP asset to set its `ReplayEditorWidget` default") still fails. Repro: (1) `asset.search query="*" parentClassPath="/Script/App.DroneRacingTrack" limit=10` → returns only `B_Race_C_1` (a World Partition external-actor instance at `/App/ChemicalPlantEnv/Maps/.../PersistentLevel.B_Race_C_1`), NOT `/App/App/LevelBlueprints/B_RaceTrack.B_RaceTrack`. (2) `asset.search query="B_RaceTrack" parentClassPath="/Script/App.DroneRacingTrack" path="/App/App" limit=20` → `{"assets":[],"count":0}`. (3) `asset.search query="B_RaceTrack" classFilter="Blueprint"` confirms the BP asset at `/App/App/LevelBlueprints/B_RaceTrack.B_RaceTrack` with `class: "Blueprint"` exists in the registry. Expected: the filter should match Blueprint assets via their `ParentClass`/`GeneratedClass` tag (as the ticket proposal described). Actual: `FARFilter::ClassPaths + bRecursiveClasses=true` matches only assets whose *own class* is in DroneRacingTrack's subclass tree — a BP asset's own class is `Blueprint`, so it is never matched. Cross-check: `blueprint.list class="DroneRacingTrack" pathStartsWith="/App/App/LevelBlueprints"` → `{"blueprints":[],"count":0}`, so the claim that "blueprint.list verified to already work correctly" does not hold against this project's asset. Mutual exclusion check passed (`classPathFilter + parentClassPath` → `INVALID_ARGUMENT`). Fix likely needs the Blueprint-specific `GeneratedClass`-tag walk (e.g., `IAssetRegistry::GetDerivedClassNames` + match on `ParentClass` AssetData tag), not the raw `FARFilter` class-paths recursive.
- `#4-bp-parent-class-tag-walk` `IN-REVIEW` developer — `asset.search` now unions the existing recursive FARFilter with a Blueprint-parent-class walk: calls `IAssetRegistry::GetDerivedClassNames` on the resolved native class and matches BP-family assets whose `FBlueprintTags::ParentClassPath` tag resolves into that set. Catches `Blueprint` / `WidgetBlueprint` / `AnimBlueprint` assets that previous FARFilter-only logic skipped because a BP asset's own class is `/Script/Engine.Blueprint`, not the native parent. Same fix applied to `blueprint.list`'s native-class branch. Pinned by `FAssetSearchNativeSubclassFindsBlueprintAssetTest` in `TestAssetSearchNativeSubclassLive.cpp`. Counterfactual: reverting the parent-tag walk makes `asset.search parentClassPath="/Script/Engine.Actor"` return zero `Blueprint`-class rows for a transient `UBlueprint` with `ParentClass=AActor`; the assertion fails.
- `#5-verified-bp-parent-tag-walk` `DONE` tester — Verified live: `mcp__editor_automation__.call path="asset.search" args={"query":"B_RaceTrack","parentClassPath":"/Script/App.DroneRacingTrack","limit":10}` now returns the actual BP asset `/App/App/LevelBlueprints/B_RaceTrack.B_RaceTrack` (class `Blueprint`) at index 0, plus 8 sibling tracks (B_RaceTrackTraining/02-07, B_RaceTrack_Stabilized_Trainig_01) all class `Blueprint`. The exact case the prior tester returned for is now answered; the BP asset is no longer skipped because of the FARFilter-only own-class match.
- `#6-additional-search-classes-inmemory-only` `DONE` reporter — Additional evidence (same "enumerate BP subclasses of a class" need, observed on a different entry method). This session needed every gate Blueprint deriving from `B_GateBase` — itself a BP (generated class `/App/App/LevelBlueprints/Gates/B_GateBase.B_GateBase_C`) — and worked around it with a `Glob` over the Gates content folder. Two corrections vs. the proposed framing: (1) `system.inspect.search_classes` is **loaded-only**, not "BP `_C` excluded" — it iterates `TObjectIterator<UClass>` and filters with `Cls->IsChildOf(ParentClass)` (`ClassSearchHandler.cpp:87,171`), so loaded BP `_C` classes *do* match; the real gap is that unloaded BP subclasses aren't in memory and it never queries the asset registry. (2) The underlying need is already served by THIS ticket's shipped fix and covers BP parents, not only native ones: `asset.search parentClassPath="/App/App/LevelBlueprints/Gates/B_GateBase.B_GateBase_C"` resolves the BP class via `ResolveUClass` (`ClassUtils.cpp:115-146`, explicitly handles content-mount BP paths with/without `_C`) and enumerates subclasses through `AppendBlueprintAssetsDerivedFromNativeClass` (`AssetUtils.cpp:1200-1257`), which is class-kind-agnostic (`GetDerivedClassNames` + `FBlueprintTags::ParentClassPath` tag walk). So this is a discoverability gap on `search_classes` (and a Glob workaround that was unnecessary), not a missing capability — no new ticket filed. Ticket stays `DONE`.
