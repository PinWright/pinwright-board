---
id: B-asset-import-unconditional-replace-hidden-outputs
title: "asset.import unconditionally replaces existing assets and reports only the first object produced by a potentially multi-output import"
status: OPEN
severity: High
category: bug
tags: [asset, import, overwrite, data-loss, partial-result, false-success]
---

# `asset.import` can overwrite content without opt-in and hides collateral outputs

## What's wrong

`AssetManageHandler.cpp:290-297` always sets `UAutomatedAssetImportData::bReplaceExisting = true`
before calling `ImportAssetsAutomated`. The schema has no `overwrite`/`replace` parameter and the
handler never checks whether the destination package already exists. A repeated import can
therefore replace a hand-edited asset with no refusal, backup, or warning.

The same call can return several objects, but `:299-308` selects the first non-null object and
discards every other identity. It then optionally renames that one object and publishes one
`assetPath` (`:310-342`). A factory that creates multiple assets can overwrite or create more
packages than the response admits. If the rename fails, the response is still success and the
published path is the requested path even though destination verification is optional.

## What it should do

Default to `overwrite:false` and refuse every pre-existing target before replacement. Return a
measured `results[]` for every imported object, including its canonical path, whether it replaced
something, rename outcome, and persistence state. An explicit overwrite must preserve or stage
the old package until all outputs are verified.

## Workaround

Import into a new empty folder and use `asset.list` to discover all produced assets before moving
them into final locations.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the data-loss and false-success catalogs; no editor, build, test, or RPC run was performed.
