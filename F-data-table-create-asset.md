---
id: F-data-table-create-asset
title: "No RPC to create a DataTable asset — data_table.* is row-CRUD only, asset.* create is folder/material only"
status: OPEN
severity: Medium
category: feature
tags: [data-table, authoring, asset-create-gap, loot-table]
encounters: 1
lastSeen: 2026-07-02T04:48:38+03:00
---

# No RPC to create a DataTable (loot table) asset

There is no way to create a `UDataTable` via the MCP. The `data_table.*`
namespace (`list_rows`/`add_row`/`set_row`/`remove_row`/`describe`, shipped by
`F-data-table-row-authoring`) can only mutate rows of a table that **already
exists** — every verb resolves an existing `UDataTable*` and bails
`ASSET_NOT_FOUND` / `NOT_A_DATATABLE` if the asset is not there. The only
asset-creating verbs elsewhere are `asset.create_folder` /
`asset.create_material` / `asset.create_material_instance`. So a caller
following the natural "give the chest a loot table" flow has no first step: the
table itself cannot be authored.

## Process friction (why this is a struggle, not just a gap)

In the task that surfaced this, the goal asked the chest to "pull its rewards
from a loot table". `interaction.configure_chest_properties` accepts a
`lootTablePath`, which invites the caller to point at a table — but there is no
verb to produce one. The agent spent ~5 discovery reads/greps confirming the
dead end (`Grep 'loot'`, `Grep 'DataTable|data.?table|RowStruct'`,
`Grep 'create_data_table|create_datatable|create_table|import_data_table|create_asset'`,
`Read data_table.md`, `Read asset.md`) before concluding "No direct
DataTable-create RPC exists" and leaving `lootTablePath` unset — an unfinished
part of the requested deliverable, reached only after wasted search.

**Fix:** add a `data_table.create(name, savePath, rowStruct)` convenience RPC
(construct the `UDataTable` via `IAssetTools::CreateAsset` with
`UDataTableFactory`, bind `RowStruct` to the supplied struct path, then
`MarkPackageDirty`). This closes the loop with the existing row CRUD (create
table -> add_row -> set `lootTablePath`). Secondarily,
`docs/wiki-src/data_table.md` and `docs/wiki-src/asset.md` should state up front
that table creation is not currently supported and name the intended path, so
the next caller does not repeat the 5-grep discovery dead end.

severity rationale: impact=missing verb with no workaround (creating a DataTable is impossible via MCP) × reach=rare path (loot-table authoring; bump down one from the missing-verb High) -> Medium.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an interaction-authoring task whose goal required a loot table. `interaction.configure_chest_properties` accepts `lootTablePath`, but no MCP verb creates a `UDataTable`: `data_table.*` is row-CRUD on an existing table (bails ASSET_NOT_FOUND otherwise) and `asset.*` create is folder/material/material-instance only. Agent ran ~5 greps/reads (`loot`, `DataTable|RowStruct`, `create_data_table|create_datatable|create_table|import_data_table|create_asset`, `data_table.md`, `asset.md`) to confirm the gap, then left `lootTablePath` unset. `F-data-table-row-authoring` (DONE) added the row surface but explicitly does not create the table asset. Proposed `data_table.create(name, savePath, rowStruct)`.
