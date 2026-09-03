---
id: B-data-table-mutators-no-persistence-report
title: "DataTable row CRUD and destructive struct rebind only mark the table dirty, then report completed mutations without persistence state"
status: OPEN
severity: High
category: bug
tags: [data-table, persistence, row, set-row-struct, no-disk-write, false-success]
---

# DataTable mutations vanish on cold load unless the caller guesses a save is required

## What's wrong

The DataTable writer family ends with `MarkPackageDirty` and a success response, never a disk write
or persistence report:

- `data_table.add_row`: dirty helper at `DataTableAuthoringHandler.cpp:306`, success at `:504-510`.
- `data_table.set_row`: `:606-612`.
- `data_table.remove_row`: `:646-652`.
- `data_table.set_row_struct`: clears every row, rebinds the struct, dirties, and succeeds at
  `:694-707`.

The sibling `data_table.create` description explicitly says it leaves the package dirty and tells
the caller to use `asset.save` (`:711-721`); none of these mutation schemas or results carries
that warning. Concrete failure: remove a row or force a destructive struct rebind, receive success,
restart without saving, and the old table returns from disk.

## What it should do

Give the family a uniform `save` parameter and standard measured save report. A caller-selected
dirty-only mutation must return the pending state explicitly, including on destructive rebind.

## Workaround

Call `asset.save` on the DataTable after every row or struct mutation.

## History
- `#1-pattern-scan` `OPEN` reporter — Grouped because all four verbs share one DataTable persistence contract. Source only; no editor, build, test, or RPC run.
