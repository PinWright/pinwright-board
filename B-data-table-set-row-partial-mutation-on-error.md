---
id: B-data-table-set-row-partial-mutation-on-error
title: "data_table.set_row deserializes directly into an existing row, so a late conversion error leaves earlier fields mutated despite an error response"
status: IN-REVIEW
severity: High
category: bug
tags: [data-table, set-row, rollback, partial-mutation, json, false-failure]
---

# A failed `data_table.set_row` can still change the row

## What's wrong

For an existing row, `data_table.set_row` calls `Table->Modify` and then passes the live row memory
directly to `FJsonObjectConverter::JsonObjectToUStruct`
(`DataTableAuthoringHandler.cpp:589-603`). On conversion failure it broadcasts post-change and
returns `INVALID_ROW_VALUES`; it does not restore the row.

UE 5.8's converter iterates properties, writes each successful value into the supplied container,
and returns false immediately when a later property fails (`JsonObjectConverter.cpp:1322-1367`).
Concrete path: send an object whose first field is valid and whose later enum/object field is
invalid. The call errors, but the first field remains changed in the live DataTable. A later save
can persist a mutation the caller was told failed.

`B-data-table-row-values-silent-drop` fixed unmatched-key handling; its history explicitly leaves
the matched-key `INVALID_ROW_VALUES` branch untouched.

## What it should do

Deserialize into initialized scratch row memory and commit it only after the complete conversion
succeeds, or snapshot and restore the exact row on every failure. Do not broadcast/dirty until
the commit point.

## Workaround

After any `INVALID_ROW_VALUES`, reload or rewrite the full row from a known-good snapshot before
saving the table.

## Fix

`data_table.set_row` now copies the existing row into initialized scratch memory, applies the
JSON conversion there, and only after success calls `Modify`/change notifications and copies the
validated row back to the live allocation. A late conversion error therefore leaves the live row
and package state untouched. Added automation coverage in
`PinWright.data_table.set_row.AtomicOnLateConversionFailure` with a valid early-field update and
an invalid late enum, asserting `INVALID_ROW_VALUES` and byte-identical row storage.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation against UE 5.8 converter code; no editor, build, test, or RPC run was performed.
- `#2-atomic-scratch-row` `IN-REVIEW` developer — Changed `DataTableAuthoringHandler.cpp` to validate existing-row values in scratch memory before the commit point; added the late-conversion byte-identity regression test. Unreal/build/automation verification was not run per task brief.
