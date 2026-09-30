---
id: B-asset-save-skips-never-saved-clean-package
title: "asset.save (no force) on a new never-saved MaterialInstanceConstant reports saveState:failed via SkippedAlreadyClean - the package exists only in memory but is not flagged dirty, so the unforced save writes nothing"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, save, material-instance, create_material_instance, pie, dirty-flag, skipped-already-clean]
encounters: 1
lastSeen: 2026-09-29T19:00:00Z
---

# asset.save skips a never-saved, not-dirty package and calls it `failed`

## Symptom

During PIE, 16 new assets were created in `/App/MultiplayerLevelEditor/PostProcess/`:
one master via `material.compile_mgir` (`save:true`, which reported `saveState:blockedByPie`)
and 15 `UMaterialInstanceConstant`s via `material.authoring.create_material_instance`
(`save:false`, all with inline `parameters.scalar`, some with `parameters.vector`).
After `editor.stop`, an unforced `asset.save {assetPath}` was called on each one:

- 11 returned `saveState:"written", saved:true`.
- 5 MIs (`MIPP_Pat_P1`, `_P4`, `_P5`, `_P3c8`, `_P4c6`) returned
  `saved:false, pendingFlush:true, saveState:"failed", sizeBytes:0`, and a retry did the same.

The editor log shows why:

```
LogPinWrightSubsystem: Warning: SaveAssetToDiskReportingPresence: '/App/MultiplayerLevelEditor/PostProcess/MIPP_Pat_P1'
is NOT durable after this save. state=failed outcome=SkippedAlreadyClean forced=false dirtyBefore=false
existedBefore=false sizeBefore=-1 ...
```

So the package had never been written (`existedBefore=false`) but was not marked dirty, and the
unforced path skipped it. `editor.list_dirty_packages` also returned `count:0` at that point, so
those 5 assets would be silently lost on editor quit. `asset.save {force:true}` wrote all five
(`saveState:"written"`).

No obvious pattern: the created MIs were identical in shape (same parent, same scalar params);
which ones ended up clean looks timing-dependent.

## Expected

- A package with no file on disk (`existedBefore=false`) is never "already clean": the unforced
  save should write it (or treat it as dirty).
- `create_material_instance {save:false}` should leave the new package dirty so
  `editor.list_dirty_packages` and `editor.save_all` pick it up.
- If a skip does happen, `saveDetail` should say "skipped: package not dirty; pass force:true",
  not the generic "a flush will not help until the cause is cleared".

## Repro

1. `editor.play`.
2. `material.authoring.create_material_instance {name, parentMaterial:<any master>, path, save:false, parameters:{scalar:{X:1}}}` x ~15.
3. `editor.stop`.
4. `asset.save {assetPath}` on each (no force) - some return `failed` with `SkippedAlreadyClean`.

## History

- `#1-reported-skipped-clean` OPEN (Reporter): observed on the PDS unreal-fpv checkout while building
  selection-outline prototype assets; workaround is `force:true`.
- `#2-re-rated` `OPEN` triage — Severity Medium -> High. Impact class is Critical (a created asset is lost: never written, not flagged dirty, `editor.list_dirty_packages` reports 0, so `editor.save_all`/quit silently drop it), bumped down one for reach because it was observed only for assets created during PIE and is timing-dependent.
- `#3-fixed-never-saved-dirty` `IN-REVIEW` (Developer): Root cause (mechanism confirmed in engine source, the live PIE repro not re-run): `UObjectBaseUtility::MarkPackageDirty` silently refuses while `GIsPlayInEditorWorld` is set (and during transactions/loads), so a create verb dispatched while a PIE world was the tick context left its new package clean; `SaveLoadedAssetThrottled` then took the `bOnlyIfIsDirty` no-op and reported `SkippedAlreadyClean`, and `FEditorFileUtils`' dirty lists (list_dirty_packages, save_all, quit prompt) never saw it. Fix: new `MarkNeverSavedPackageDirty(UPackage*)` in `Source/PinWright/Private/Utils/AssetUtils.cpp` sets the dirty flag on a clean, mounted, non-transient, non-PIE, non-/Temp package whose `.uasset`/`.umap` does not exist. Called (1) at the top of `SaveLoadedAssetThrottled`, ahead of the PIE gate and throttle, so an unforced save writes a never-saved package (or reports `blockedByPie`/`deferred` and leaves it dirty), and (2) from a new `IAssetRegistry::OnInMemoryAssetCreated` hook in `UPinWrightSubsystem::Initialize` (`PinWrightSubsystem.cpp`/`.h`), which covers every create verb that calls `FAssetRegistryModule::AssetCreated` without a per-verb change. The "skipped: pass force:true" saveDetail wording is moot: a never-saved package no longer reaches the skip. Docs: `docs/wiki-src/safe-mutation-save.md` (Save States). Tests: `PinWright.assets.AssetSaveState.NeverSavedCleanPackageIsWritten` (clean never-saved package, unforced save must report `written` and put the `.uasset` on disk; pre-fix `failed`), `PinWright.assets.AssetSaveState.CreatedCleanAssetIsListedDirty` (MarkPackageDirty refused under `GIsPlayInEditorWorld`, then `AssetCreated` must leave the package dirty and in `GetDirtyContentPackages`; pre-fix clean).
