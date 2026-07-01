---
id: B-asset-dump-redirector-not-followed
title: "asset.dump_folder includes UObjectRedirector assets as if they were real, with empty properties.json"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, redirector]
---

# asset.dump_folder includes UObjectRedirector assets as if they were real, with empty properties.json

An ObjectRedirector ends up in the dump cache as if it were a real asset. Folder name suggests a DataAsset, but `meta.json` reports `className: ObjectRedirector` and `properties.json` is empty `{}`.

A consumer reading the cache and resolving the asset gets a confusing redirector record instead of being routed to the redirected target.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/DA_AppMusicConfig/`.
2. Observe: `meta.json` reports `className: ObjectRedirector`, `properties.json` is `{}`.

**Fix (proposed):** `BuildAssetMeta` (or the folder traversal filter) doesn't skip `UObjectRedirector` instances. Either skip redirectors during dump_folder, or follow the redirector to its target and emit that data with a `redirectedFrom` note in meta.

## History
- `#1-initial-repro` `OPEN` reporter — UObjectRedirector dumps are emitted as if real, with `className: ObjectRedirector` in meta and `{}` in properties. Sample path: `App/App/DA_AppMusicConfig/`. Consumer cannot follow the redirect from the cache alone.
- `#2-dump-redirectors-as-redirectors` `IN-REVIEW` developer — Special-cased `UObjectRedirector` in `Utils/AssetDumpBuilder.cpp::BuildMetaJson` (new `redirectsTo` field) and `Handlers/Asset/AssetDumpHandler.cpp::BuildAllFilesForAsset` (properties.json emits `{ redirectsTo: <target path> }` instead of empty `{}`). Redirectors are dumped at their original path; the redirect target is dumped separately under its own path. Added `Tests/Private/Utility/TestAssetDumpRedirector.cpp` (`FAssetDumpRedirectorEmitsRedirectsToTest`).
- `#3-review-fixes-extract-helper` `IN-REVIEW` developer — Phase 4.5 cleanup: extracted `AssetDumpBuilder::ResolveRedirectorTarget(UObjectRedirector*)` helper, dedup'd target-path resolution between `BuildMetaJson` and `BuildRedirectorPropertiesAspect_Internal`.
- `#4-verify-fix` `DONE` tester — Verified: `asset.dump` on `/App/App/DA_AppMusicConfig` regenerated cache files; `meta.json` now contains `redirectsTo: "/Game/Audio/FPV_SOUND/Music/DA_AppMusicConfig.DA_AppMusicConfig"` (className still `ObjectRedirector`, dumpSchemaVersion bumped to 5), and `properties.json` now contains `{ "redirectsTo": "/Game/Audio/FPV_SOUND/Music/DA_AppMusicConfig.DA_AppMusicConfig" }` instead of empty `{}`.
- `#5-audit-all-redirects-in-cache` `DONE` reporter — Audit of current asset-dump cache found exactly 5 `redirectsTo` entries (full list): `App/App/DA_AppMusicConfig` → `/Game/Audio/FPV_SOUND/Music/DA_AppMusicConfig.DA_AppMusicConfig`; `Game/Maps/New_InfinityMap/SM_GridFloor_10x10m` → `/Game/Maps/New_InfinityMap/SM_GridFloor_10x10m_Textured.SM_GridFloor_10x10m_Textured`; `Game/Maps/New_InfinityMap/M_Grid1` → `/Game/Maps/New_InfinityMap/M_Grid_Textured.M_Grid_Textured`; `Game/Maps/New_InfinityMap/MI_Grid` → `/Game/Maps/New_InfinityMap/MI_Grid_Procedural.MI_Grid_Procedural`; `Game/UI/Foundation/LoadingScreen/W_LoadingScreen_DefaultContent` → `/App/App/UI/LobbyAndMenu/LoadingScreen/W_LoadingScreen_DefaultContent.W_LoadingScreen_DefaultContent`. Contrary to a prior assumption that 4 of the 5 target paths were missing, on-disk and in-cache verification shows **all 5 redirect targets exist** both as live `.uasset`/`.umap` files (under `Content/` and `Plugins/App/Content/`) and as dumped folders under `.editor-automation/asset-dumps/`. No dangling-target bug observed in the current cache. The fix from #2/#3 (emit `redirectsTo` from `Redirector->DestinationObject->GetPathName()` in `AssetDumpBuilder::ResolveRedirectorTarget`) is working — `DestinationObject` was non-null for every redirector encountered, so every emitted target resolves. Note: the emitter still has no defensive validation that the target asset is loadable/registered; if a future redirector ever has `DestinationObject == nullptr` it emits an empty-string `redirectsTo`, and if `DestinationObject` points to a stale object the path string could in principle reference an unregistered asset. Neither failure mode is reproducing today. No re-open warranted; filing as audit note.
