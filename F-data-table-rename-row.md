---
id: F-data-table-rename-row
title: "No rename verb for a DataTable row — renaming a row forces a remove_row + add_row struct round-trip that hand-carries every field"
status: OPEN
severity: Low
category: feature
tags: [data-table, authoring, row-map, rename, lifecycle, remove-add-fallback]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
---

# No rename verb for a DataTable row

The `data_table` namespace (shipped by the DONE `F-data-table-row-authoring`) is
**add / set / remove only** for rows. Its verb roster per the wiki overlay
(`docs/wiki-src/data_table.md`) and the shipped handlers is exactly
`describe`, `list_rows`, `add_row`, `set_row`, `remove_row`, `set_row_struct` —
**no `rename_row`**. A `UDataTable` row is keyed by its `FName` in `RowMap`, so
changing only a row's *name* (a pure key rename, struct values unchanged) has no
typed verb. The caller must reconstruct it as **remove + add**, hand-carrying the
old row's struct payload across the two calls.

## Process friction (this is the PROCESS angle — outcome already judged)

The audited `DT_Colourways` brand-palette task hit this directly. Requirement (4):
*"rename our greys consistently — since there's no rename verb, replace
`DefaultGrey` by removing it and adding `NeutralGrey` with the same
`{r:108, g:108, b:108, a:255}` values."* The task's own story narrates the
workaround in-line because the surface offers nothing better:

- `data_table.remove_row` → drop `DefaultGrey`.
- `data_table.add_row` → recreate as `NeutralGrey` with the values **re-typed by
  hand** (`{r:108, g:108, b:108, a:255}`).

That is **2 calls plus a manual value carry** for a one-line intent ("rename
`DefaultGrey` to `NeutralGrey`"). Friction note read "none" only because the agent
followed the pre-scripted remove+add the story already spelled out — but the
remove+add *is* the friction: a single `rename_row(DefaultGrey, NeutralGrey)` would
collapse it to one call with zero value re-entry and zero risk of a transcription
slip in the re-typed struct. The cost scales with struct width — `S_Colour` has 4
fields here; a wide row struct multiplies the hand-carry surface, and any field the
caller forgets to copy is silently lost (set to the struct's default) because
`add_row` builds a fresh row.

This is the same **missing-rename-verb** class already filed for montage sections
(`F-montage-section-remove-rename`: "no `rename_montage_section`… the agent had to
drop to a struct-array round-trip") and noted for widgets
(`E-widget-add-then-rename-discoverability`). DataTable rows are the row-map
analogue: the rename is conceptually a key change, but the only path is
destroy-and-rebuild-the-value.

## What it should do

Add a symmetrical lifecycle verb mirroring the existing `add_row` / `remove_row`:

- `data_table.rename_row(assetPath, rowName, newName)` — rename the row's `FName`
  key **in place**, preserving the existing struct allocation (not a
  copy-via-JSON), routed through `FDataTableEditorUtils::RenameRow` (the same path
  the editor's row-name edit uses, which calls `Modify` and fires
  `OnDataTableChanged` / refreshes asset-registry tags). Error `ROW_NOT_FOUND` if
  `rowName` is absent and `ROW_EXISTS` if `newName` already exists (reuse the same
  error vocabulary `add_row` / `remove_row` already emit, observed verbatim in this
  task as `[ROW_NOT_FOUND]` / `[ROW_EXISTS]`). Return the updated row via the same
  `DataTableDumpBuilder::BuildRowJson` helper the other verbs use so the readback
  shape stays byte-identical.

Implementation surface: `FDataTableEditorUtils::RenameRow(UDataTable*, OldName,
NewName)` already exists in UnrealEd and does exactly this (in-place key change with
the correct editor invariants) — co-locate the handler in the existing
`Handlers/DataTable/DataTableAuthoringHandler.cpp` next to `add_row` / `remove_row`,
no new dependency. Document the verb in `docs/wiki-src/data_table.md` (extend the
one-line roster to include rename).

**Workaround (current):** `data_table.remove_row(oldName)` +
`data_table.add_row(newName, <values hand-re-typed from the old row>)`. 2 calls,
manual struct value carry, silent field loss if any field is omitted.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle (process) audit of the
  `DT_Colourways` brand-palette refresh task (namespace data_table, outcome clean,
  12 calls). Requirement (4) needed a row rename `DefaultGrey`→`NeutralGrey`; the
  `data_table` surface (DONE `F-data-table-row-authoring`) ships
  `describe/list_rows/add_row/set_row/remove_row/set_row_struct` but **no
  `rename_row`**, so the agent did `remove_row(DefaultGrey)` + `add_row(NeutralGrey,
  {r:108,g:108,b:108,a:255})`, hand-re-typing the struct values across the two
  calls — the story even narrates "since there's no rename verb, replace by removing
  it and adding". 2 calls + manual value carry for a one-line key rename; any
  omitted field is silently defaulted because `add_row` builds a fresh row. Same
  missing-rename-verb class as `F-montage-section-remove-rename` (sections) and
  `E-widget-add-then-rename-discoverability` (widgets). Proposes
  `data_table.rename_row(assetPath, rowName, newName)` over
  `FDataTableEditorUtils::RenameRow` (in-place key change, correct editor
  invariants, `ROW_NOT_FOUND`/`ROW_EXISTS` errors matching the existing vocabulary),
  co-located in `DataTableAuthoringHandler.cpp`, documented in the
  `docs/wiki-src/data_table.md` overlay.
