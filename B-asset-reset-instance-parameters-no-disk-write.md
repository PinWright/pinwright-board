---
id: B-asset-reset-instance-parameters-no-disk-write
title: "asset.reset_instance_parameters clears all overrides in memory, reports success, and exposes no persistence state"
status: IN-REVIEW
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

## Fix

The handler cleared overrides and dirtied the resident package but never attempted a disk write or measured the result. `AssetMaterialHandler.cpp` now defaults `save` to true, routes the reset through `SaveAssetToDiskReportingPresence`, returns the canonical asset/package identity and post-clear per-kind `remainingOverrideCounts`, and publishes the shared `AssetSaveState` report; `save:false` follows the shared `notRequested` contract and leaves the package dirty. `Dispatch/SafePoint.cpp` routes both parameter branches outside `UWorld::Tick` because the dispatcher cannot know whether the synchronous save path will run. `TestAssetMutationDurability.cpp` adds `PinWright.asset.reset_instance_parameters.DurableSaveRoundTrip`, which drives the registered handler, proves `save:false` is reverted by disk reload, proves a default-saved reset survives reload, and pins the safe-point route. `docs/wiki-src/asset.md` documents the new parameter and response fields. Deliberately unchanged: reset scope still matches Unreal's `ClearParameterValuesEditorOnly`; base-property overrides are not parameter values and remain outside this verb.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the persistence catalog; no editor, build, test, or RPC run was performed.
- `#2-durable-save-contract` `IN-REVIEW` developer — Added default durable saving, explicit save opt-out, post-reset override counts, shared save-state reporting, behavioral disk-reload coverage, and wiki documentation; source/static checks only, with build and automation deferred to the checkpoint agent.
