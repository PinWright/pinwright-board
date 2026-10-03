---
id: B-asset-audit-doc-denies-material-instance-sidecar
title: "asset-audit.md says material_instance.json is NOT an asset.dump sidecar, but the sidecar is registered and asset.md documents it"
status: OPEN
severity: Low
category: bug
tags: [docs, asset-dump, material-instance, wiki-src, stale-doc]
rice: [1, 2, 1, 1]
priority: 17
---

# The audit page contradicts the dump code and asset.md

`docs/wiki-src/asset-audit.md:49` states: "`material_instance.json` is NOT a sidecar -
`DumpFileNames` has no entry and `BuildAllFilesForAsset` has no `UMaterialInstance` branch.
Material instances fall through to the default tail and emit only `properties.json`."

That is no longer true:

- `DumpFileNames::MaterialInstance = "material_instance.json"` (`Handlers/Asset/AssetDumpHandler.h:39`).
- `REGISTER_DUMP_JSON_SIDECAR(TEXT("material_instance"), DumpFileNames::MaterialInstance, ...)`
  for `UMaterialInstanceConstant` (`Handlers/Asset/MaterialInstanceDumpBuilder.cpp`, end of file).
- `material_instance.json` has an aspect-version row in `Handlers/Asset/AssetDumpCache.cpp`.
- `docs/wiki-src/asset.md` (schema summary) documents the sidecar and its fields.

An agent reading the audit page concludes a MIC dump has no structured override sidecar and
falls back to `properties.json` or a live RPC.

## What it should do

Rewrite the `asset-audit.md:49` paragraph: `material_instance.json` is the MIC sidecar, built by
the same `MaterialInstanceDumpBuilder::BuildMaterialInstanceJson` that serves
`material.authoring.get_material_instance_info` (so the two agree field for field, including
`orphanedOverrides` / `orphanedOverrideCount`). Keep the "do not add `material_instance.describe`"
guidance if it still holds.

## Related

- `B-asset-dump-mic-no-material-instance-json` (DONE) - the change that added the sidecar.
- `E-dump-rpc-parity`

## History
- `#1-filed-from-g13-review` `OPEN` reviewer - Found while reviewing the G13 material change
  (`E-material-instance-info-orphaned-overrides`), which extends this sidecar; source-read only,
  no editor run.
