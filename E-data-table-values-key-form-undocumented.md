---
id: E-data-table-values-key-form-undocumented
title: "data_table.md wiki overlay never states the values-key form for add_row/set_row (display name, first letter lowercased, spaces preserved), so a camelCase mis-guess only fails after a round-trip"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [docs, data-table, wiki, add_row, set_row, values, key-form, user-defined-struct, discovery]
---

# `data_table` wiki never documents the `values` key spelling rule

`data_table.add_row(assetPath, rowName, values)` and
`data_table.set_row(assetPath, rowName, values)` apply the `values` object to the
row's `RowStruct` via JSON→struct conversion. For a `UserDefinedStruct` row the
accepted JSON key for each field is **not** the obvious camelCase name — it is
the field's **display name with the first letter lowercased and the spaces
preserved** (e.g. `roof Drop`, `enclosed Right`, `wall Drop`, and even the
double-spaced `sub  Pillar`). The `data_table.md` wiki overlay says nothing about
this. The runtime now **rejects** a non-matching key — the companion bug
`B-data-table-row-values-silent-drop` landed its fix (IN-REVIEW), so an unmatched
`values` key returns an `INVALID_PARAMS` error carrying `droppedFields` (the keys
that matched no field) and `validFields` (the struct's authored field names)
*before* any table mutation. That surfaces the trap, but only **after** the
caller has already authored and sent a wrong-key payload and eaten a round-trip;
the discovery cost still lands on the caller up front because the overlay never
states the key-form rule so the caller can get it right the first time.

The overlay `docs/wiki-src/data_table.md` is currently a **single sentence**:

> "Author and inspect `UDataTable` rows — describe the table's row struct and
> current contents, list rows, add/set/remove rows, and set rows from a typed
> struct payload. Use `call("property.set")` ... this namespace is scoped to
> row-content CRUD."

It never tells the caller (a) what shape the `values` object takes, (b) that the
keys are display-name-derived rather than the struct's internal variable names,
or (c) how to discover the exact emitted key form before writing. So the natural
first guess — camelCase off the field's variable name — is now bounced with an
`INVALID_PARAMS` `droppedFields`/`validFields` error, but only after the caller
has authored and round-tripped a wrong payload.

This is the **discovery / docs** angle, distinct from and complementary to the
runtime bug `B-data-table-row-values-silent-drop` (whose fix landed: the tool now
*rejects* unmatched keys with `droppedFields`/`validFields` rather than silently
dropping them). A caller who knows the key form *up front* never triggers the
reject path at all — the docs fix prevents the friction, the bug fix only
surfaces it (and now hands back the valid key set in-band on the rejected call).
It also pairs with `E-asset-dump-userdefinedstruct-field-list` (DONE), which made
`asset.dump`'s `user_defined_struct.json` sidecar a place to read the field list
— but note the sidecar emits `name` (the internal GUID-suffixed var name) and
`displayName` (the friendly name, e.g. "Roof Drop") **separately**; neither field
is the literal JSON key `roof Drop`, so the caller still has to apply the
lowercase-first-letter transform to `displayName` to build the key. The
`data_table` overlay points at neither, so the caller has to know to go look.

## Process friction (this task)

DataTable migration task (focus `data_table.set_row_struct`): rebind
`DT_Colourways` from `S_Colour` to `S_RoomSettings`, then re-populate. The
friction note (verbatim):

> "UserDefinedStruct JSON keys are the display names with spaces preserved and
> first letter lowercased (e.g. `sub  Pillar` double-space, `enclosed Right`,
> `roof Drop`), not camelCase as I first guessed, so my first add_row silently
> dropped several fields (I only saw it via readback)."

(That note describes the *pre-fix* runtime — the silent drop. With
`B-data-table-row-values-silent-drop` fixed, the same wrong-key `add_row` now
returns an `INVALID_PARAMS` error listing `droppedFields`/`validFields` instead of
silently no-opping, so the caller sees it on the first call rather than via a
readback. But it is still a *failed* first call: the caller authored the wrong key
form, round-tripped, and had to re-author. Documenting the rule removes that
wasted round-trip entirely.)

Even with the in-band `validFields` from the reject path, the underlying cost is a
wrong-key first attempt the docs would have prevented: the documented key-form
rule plus a "derive the key from the struct's `displayName`" pointer in the
overlay would have collapsed that to zero discovery detours.

## What it should do / how to fix

Docs-only (downstream wiki process — not a code change): expand
`docs/wiki-src/data_table.md` with a short authoring block for `add_row` /
`set_row` that states the `values` key rule and the discovery path, e.g.

> **`values` key form.** Each key in `values` is the row struct **field's display
> name with its first letter lowercased and internal spaces kept verbatim** —
> e.g. a field shown as "Roof Drop" is keyed `"roof Drop"`, "Enclosed Right" is
> `"enclosed Right"`. This is *not* camelCase: `roofDrop`/`RoofDrop` will not
> match and (today) are silently dropped. Discover the exact keys first with
> `asset.dump` on the struct (read the `user_defined_struct.json` sidecar) or a
> `data_table.list_rows` / `data_table.describe` readback, and copy the emitted
> key form verbatim. After writing, diff the echoed `row` against what you sent.

```jsonc
// add_row against a row struct with displayed fields "Room Name", "Roof Drop"
{ "assetPath": "/Game/.../DT_Rooms", "rowName": "Studio",
  "values": { "roomName": "Studio", "roof Drop": 0.5, "wall Drop": "Open" } }
```

**Workaround:** until the overlay is filled in, dump the row struct
(`asset.dump` → `user_defined_struct.json`) or `list_rows` an existing row to
read the exact key spellings before authoring `values`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of a DataTable migration task (focus `data_table.set_row_struct`). The `data_table.md` wiki overlay (`docs/wiki-src/data_table.md`) is a single-sentence namespace blurb that never documents the `values` key form for `add_row`/`set_row`: for a `UserDefinedStruct` row the key is the display name with first letter lowercased + spaces preserved (`roof Drop`, `enclosed Right`, `sub  Pillar`), not camelCase. The caller guessed camelCase, the first `add_row` silently dropped fields (only caught on readback), and recovery took two `asset.dump` discovery calls — including one on `S_Colour` purely "to learn key casing." Distinct from the judge-filed runtime bug `B-data-table-row-values-silent-drop` (reject/echo unmatched keys): this is the up-front discoverability angle that, once documented, prevents the trap rather than surfacing it; pairs with `E-asset-dump-userdefinedstruct-field-list` (DONE) which the overlay never points at. Proposed deliverable: a `values`-key-form authoring block + "dump the struct first" pointer + tiny example in `docs/wiki-src/data_table.md`. Dedup: ripgrep across OPEN/closed found no data_table docs/key-form ticket; the only `data_table.md` mention is `F-data-table-row-authoring` #5 (which created the bare overlay) and the runtime bug above.
