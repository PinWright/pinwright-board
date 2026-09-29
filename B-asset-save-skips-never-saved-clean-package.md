---
id: B-asset-save-skips-never-saved-clean-package
title: "asset.save (no force) on a new never-saved MaterialInstanceConstant reports saveState:failed via SkippedAlreadyClean - the package exists only in memory but is not flagged dirty, so the unforced save writes nothing"
status: OPEN
severity: Medium
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
