---
id: B-asset-list-short-class-ensure
title: "`asset.list` fires `FTopLevelAssetPath` ensure on short class names"
status: DONE
severity: Medium
category: bug
tags: []
---

# `asset.list` fires `FTopLevelAssetPath` ensure on short class names

No functional failure (post-filter fallback returns correct results), but every call with `filter.class = "WidgetBlueprint"` / `"Material"` etc. fires `Ensure condition failed: false` at `CoreUObject/Private/UObject/TopLevelAssetPath.cpp:141` with message `Short asset name used to create FTopLevelAssetPath: "<ClassName>"`. Log spam and misleads developers into thinking MCP is misbehaving.

Root cause: `AssetManageHandler.cpp:718` constructs `FTopLevelAssetPath ClassPath(ClassFilter)` directly from the user-supplied filter string. UE requires the full `/Script/Package.ClassName` form; passing a short name fires the engine ensure before the `IsValid()` guard at line 719 can skip it.

**Workaround:** Use the full class path (e.g. `filter.class = "/Script/UMGEditor.WidgetBlueprint"`).

**Fix:** Before constructing `FTopLevelAssetPath`, detect short names (no `/` or no `.`) and either resolve via `UClass::TryFindTypeSlow<UClass>(ClassFilter)` to the full path, or skip the `ClassPaths.Add` branch entirely and rely on the existing post-filter `AssetClassName.Equals(ClassFilter)` path at line 744 (which already handles short names).

## History
- `#1-initial-repro` `OPEN` reporter — Surfaced during BPIR auto-formatter investigation. Called `asset.list` with `filter={"class":"WidgetBlueprint"}`; call succeeded (returned /Game widgets via post-filter fallback) but PDS.log shows two ensure fires at `TopLevelAssetPath.cpp:141` with message `Short asset name used to create FTopLevelAssetPath: "WidgetBlueprint"`. Stack trace: `AutoHandler_30_` → `FTopLevelAssetPath::FTopLevelAssetPath(FString)` → `TrySetPath`.
- `#2-resolve-uclass-fix` `IN-REVIEW` developer — `AssetManageHandler::asset.list` now routes short class names through `ResolveUClass` (`Utils/ClassUtils.h`) before constructing `FTopLevelAssetPath`, so the ensure no longer fires. Covered by `FAssetListShortClassNameNoEnsureTest` in `TestAssetHandlers.cpp`.
- `#3-verified-no-ensure` `DONE` tester — Verified with `asset.list path="/App/App/UI" filter={"class":"WidgetBlueprint"}`. Call succeeded, returned 3 WidgetBlueprint assets. `grep "Short asset name used to create FTopLevelAssetPath" Saved/Logs/PDS.log` returned no matches — no ensure fired. Previously this call would log two ensure messages.
