---
id: B-asset-reset-instance-parameters-no-disk-write
title: "asset.reset_instance_parameters clears all overrides in memory, reports success, and exposes no persistence state"
status: OPEN
severity: High
category: bug
tags: [asset, material-instance, reset, persistence, no-disk-write, false-success]
---

# Reset material-instance parameters is dirty-memory only

## What's wrong

`asset.reset_instance_parameters` calls `ClearParameterValuesEditorOnly`, `PostEditChange`, and
`MarkPackageDirty` (`AssetMaterialHandler.cpp:129-131`), then returns `success:true` and the request
path (`:133-136`). It neither writes the package nor reports that the edit is pending a save.

Concrete failure: clear all overrides, trust the success response, close or reload without a
separate save, and every override returns from the old package. The operation is destructive in
memory but non-durable on disk, and the response has no field that distinguishes those states.

## What it should do

Add a `save` parameter (default true), use the standard measured save helper/report, and return
the canonical asset path plus the observed remaining override counts. If save is false, report
`saved:false`/`pendingFlush:true` explicitly.

## Workaround

Call `asset.save` on the material instance immediately after the reset.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the persistence catalog; no editor, build, test, or RPC run was performed.
