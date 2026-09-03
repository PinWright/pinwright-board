---
id: B-asset-set-metadata-no-disk-write
title: "asset.set_metadata advertises persisted package metadata but only dirties the package and reports no save outcome"
status: OPEN
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

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the persistence/request-readback catalogs; no editor, build, test, or RPC run was performed.
