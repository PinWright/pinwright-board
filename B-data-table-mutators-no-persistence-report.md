---
id: B-data-table-mutators-no-persistence-report
title: "DataTable row CRUD and destructive struct rebind only mark the table dirty, then report completed mutations without persistence state"
status: IN-REVIEW
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

## Fix

The ticket was true: all four mutators stopped after dirtying the package and built their success response without any save request or measured persistence state. They now share one `AssetSaveState` path: `save` defaults to true, a requested mutation force-writes the DataTable package through `SaveAssetToDiskReportingPresence`, and every successful response reports `saveRequested`, `saved`, `saveState`, `saveDetail`, plus `pendingFlush` when a requested save is not durable; `save:false` keeps the edit in memory and reports `notRequested` without writing.

Files changed: `Source/PinWright/Private/Handlers/DataTable/DataTableAuthoringHandler.cpp`, `Source/PinWright/Private/Dispatch/SafePoint.cpp`, `Source/PinWright/Private/Tests/DataTable/TestDataTablePersistence.cpp`, and `docs/wiki-src/data_table.md`. Test id: `PinWright.data_table.mutators.PersistenceAndColdReload`; it invokes every mutator through the handler harness, reloads the package from disk, checks durable state, checks each `save:false` branch is discarded by reload, and pins all four verbs in the tick-unsafe table.

Deliberately unchanged: `data_table.create` remains a dirty-only bootstrap operation with its existing explicit `asset.save` guidance; this ticket covers mutations of an existing table. Per the worker brief, no build, editor, RPC, or automation run was performed here.

## History
- `#1-pattern-scan` `OPEN` reporter — Grouped because all four verbs share one DataTable persistence contract. Source only; no editor, build, test, or RPC run.
- `#2-persistence-contract` `IN-REVIEW` developer — Added uniform default-on saving and measured AssetSaveState reports to all four mutators, safe-point routed their synchronous writes, documented `save:false`, and added handler-level cold-reload coverage.
