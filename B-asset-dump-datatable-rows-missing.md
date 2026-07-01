---
id: B-asset-dump-datatable-rows-missing
title: "asset.dump DataTable dumps omit row contents"
status: DONE
severity: High
category: bug
tags: [asset, dump, datatable]
---

# asset.dump DataTable dumps omit row contents

DataTable assets currently dump only generic metadata and reflected properties. The semantic row names and row values are not emitted, so cached dumps are not useful for inspecting tables.

**Workaround:** Use a dedicated editor/Python/DataTable reader when exact row contents are needed.

**Fix:** Add a DataTable-specific dump aspect that reads `UDataTable::GetRowMap()`, emits row struct path plus stable row JSON/text, and covers a non-empty fixture in tests.

## History
- `#1-live-repro` `OPEN` reporter — `asset.dump /Game/Audio/Data/DT_SongData` writes only `meta.json` and `properties.json`. `properties.json` exposes `RowStruct=/Game/Audio/Data/S_SongData.S_SongData` but `RowsSerializedWithTags` is `[]`, `property.get RowMap` returns `PROPERTY_NOT_FOUND`, and no row-name/row-value aspect is emitted. Add a DataTable-specific dump aspect that reads `UDataTable::GetRowMap()`, emits row struct path plus stable row JSON/text, and covers a non-empty fixture so empty transient `RowsSerializedWithTags` cannot be mistaken for complete row export.
- `#2-datatable-dump-aspect` `IN-REVIEW` developer — Added `Handlers/DataTable/DataTableDumpBuilder.h+.cpp` (new builder iterating `GetRowMap()` via `FJsonObjectConverter::UStructToJsonObject`, rows sorted alphabetically); extended `DumpFileNames` namespace in `AssetDumpHandler.h` with `DataTable = TEXT("data_table.json")`; inserted `UDataTable` dispatch branch in `AssetDumpHandler.cpp::BuildAllFilesForAsset` before `UWorld` branch, added `DumpFileNames::DataTable` to `Canonical[]` array, and updated handler description string; appended `FTestAssetDumpDataTableRow` USTRUCT to `TestAssetDumpDataAssetFixture.h`; added `TestAssetDumpDataTable.cpp` with test `EditorAutomationRpcGateway.asset.dump.DataTableRows`; updated `docs/wiki/asset.md` DataTable bullet to mention `data_table.json`. Counterfactual: removing the `UDataTable` branch causes fall-through to generic else emitting only `meta.json`+`properties.json`, and the `data_table.json` existence assertion fails.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump /Game/Audio/Data/DT_SongData` now lists 3 files including `data_table.json`. File contains `rowCount=10`, `rowStruct=/Game/Audio/Data/S_SongData.S_SongData`, and an alphabetically-sorted `rows` map with each row's struct fields (`name`, `song`) populated.
