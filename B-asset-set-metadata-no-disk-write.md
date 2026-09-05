---
id: B-asset-set-metadata-no-disk-write
title: "asset.set_metadata advertises persisted package metadata but only dirties the package and reports no save outcome"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, metadata, persistence, no-disk-write, false-success]
---

# `asset.set_metadata` calls dirty memory “persisted”

## What's wrong

The registered description says metadata is “Persisted in the asset package”
(`AssetMetadataHandler.cpp:26`). The handler writes `UMetaData`, calls only
`Package->SetDirtyFlag(true)` (`:78-121`), and sends success with `updatedKeys` plus generic asset
verification (`:123-130`). There is no package write or save-state field.

Concrete failure: write metadata used by a later `asset.find_by_tag`, receive success, restart
without an extra save, and the metadata is absent because only the resident package changed.
Generic object/path verification cannot prove the new metadata reached disk.

## What it should do

Either save by default with the standard measured persistence report, or change the contract to
explicit dirty-only behavior and return `markedForSave`, `saved:false`, and `pendingFlush:true`.
Read the written keys back from the asset before responding.

## Workaround

Call `asset.save` on the asset after setting metadata.

## Fix

The handler treated `SetDirtyFlag(true)` as persistence and returned only live-object verification. `AssetMetadataHandler.cpp` now defaults `save` to true, verifies each written key against `UMetaData`, routes changed assets through `SaveAssetToDiskReportingPresence`, and publishes the shared `AssetSaveState` report plus canonical package/path data; `save:false` follows the shared `notRequested` contract and leaves the package dirty. `Dispatch/SafePoint.cpp` routes both parameter branches outside `UWorld::Tick` because the dispatcher cannot know whether the synchronous save path will run. `TestAssetMutationDurability.cpp` adds `PinWright.asset.set_metadata.DurableSaveRoundTrip`, which drives the registered handler, proves `save:false` disappears after disk reload, proves the default-saved value survives reload, and pins the safe-point route. `docs/wiki-src/asset.md` documents the new parameter and response fields. Deliberately unchanged: `asset.set_tags`, which is a separate handler despite describing itself as a convenience wrapper.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the persistence/request-readback catalogs; no editor, build, test, or RPC run was performed.
- `#2-durable-save-contract` `IN-REVIEW` developer — Added default durable saving, explicit save opt-out, metadata readback, shared save-state reporting, behavioral disk-reload coverage, and wiki documentation; source/static checks only, with build and automation deferred to the checkpoint agent.
