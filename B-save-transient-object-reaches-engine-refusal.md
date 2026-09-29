---
id: B-save-transient-object-reaches-engine-refusal
title: "Single-asset save sends an RF_Transient object in an ordinary package to the engine: engine Error line and state=failed instead of notPersistable"
status: IN-REVIEW
severity: Low
category: bug
tags: [asset-save, honesty, log-noise, data-table, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T10:21:04Z
---

# Transient object in a non-transient package reaches `SaveLoadedAsset`

`SaveLoadedAssetThrottled` (`Source/PinWright/Private/Utils/AssetUtils.cpp`) classifies an asset as
`NotPersistable` only when its **package** is the transient package or carries `RF_Transient`. An
object that is itself `RF_Transient` inside an ordinary package (e.g. `CreatePackage("/Engine/Transient/X")`
+ `NewObject<...>(Pkg, ..., RF_Transient)`) passes that gate, gets a mounted filename, and reaches
`UEditorAssetLibrary::SaveLoadedAsset`. The engine refuses it because `UObject::IsAsset()` is false
for `RF_Transient` objects (`ObjectTools::IsObjectBrowsable`, `EditorAssetSubsystem.cpp:124-146`, 1299-1303) and logs:

`LogEditorAssetSubsystem: Error: SaveLoadedAsset failed: Asset is not registered. The object '<name>' is not an asset.`

The verb then reports `saved:false`, `saveState:"failed"`, `pendingFlush:true` - advice to retry or flush
a save that can never land. Every verb routed through `SaveAssetToDiskReportingPresence` is affected; it
surfaced on the `data_table.*` mutators (12 engine errors across 9 tests in
`Saved/Logs/pw_gapwave_full_offscreen2.log`, masked until `B-suppress-log-errors-static-leaks` was fixed).
Users hit it only with an in-memory transient object in a non-transient package, which is rare.

**Fix:** extend the `NotPersistable` gate in `SaveLoadedAssetThrottled` to `Asset->HasAnyFlags(RF_Transient)`,
so the object is classified before the engine is asked.

## History
- `#1-engine-refusal-on-transient-object` `OPEN` reporter — Found while triaging tests newly exposed by the per-test `bSuppressLogErrors` reset: the `data_table.*` fixtures (`MakeTransientDataTable`, RF_Transient table in `/Engine/Transient/DT_AuthoringTest_*`) reach the engine save and log `SaveLoadedAsset failed: Asset is not registered`; the response carried `saveState:"failed"` for a never-persistable object.
- `#2-classify-transient-object` `IN-REVIEW` developer — Changed `SaveLoadedAssetThrottled` in `Utils/AssetUtils.cpp` to return `NotPersistable` when the object itself is `RF_Transient`. Regression checks: `PinWright.core.asset_save_honesty.save_loaded_asset_throttled.TransientNotPersistable` now also asserts an RF_Transient object in an ordinary package classifies `NotPersistable`; `PinWright.data_table.add_row.RoundTrip` asserts `saveState:"notPersistable"`; all 9 `data_table.*` tests stop logging the engine error. Compile-checked only; needs a suite run.
