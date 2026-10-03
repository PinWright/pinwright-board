---
id: B-eqs-authoring-verbs-saved-echoes-request
title: "`eqs.create` (fresh), `eqs.add_generator`, `eqs.add_test`, `eqs.set_test_filter` and `eqs.set_test_scoring` report `saved` = the request flag while only marking the package dirty"
status: OPEN
severity: Medium
category: bug
tags: [eqs, save, saved, false-success, mark-dirty, persistence]
encounters: 1
firstSeen: 2026-10-02
lastSeen: 2026-10-02
rice: [1, 3, 1, 1]
priority: 33
---

# EQS authoring verbs echo `save` into `saved` without writing anything

`Source/PinWright/Private/Handlers/AI/EQSHandler.cpp`: a fresh `eqs.create` (no existing asset,
`save` defaults to **true**), `eqs.add_generator`, `eqs.add_test`, `eqs.set_test_filter` and
`eqs.set_test_scoring` call `McpSafeAssetSave(Query)` — which only marks the package dirty and
notifies the asset registry — and then set `Result->SetBoolField("saved", bSave)`. So `save:true`
answers `saved:true` while nothing reached disk.

This is the same defect `B-eqs-set-context-and-blueprint-create-saved-true-no-disk-write` fixed
for `eqs.set_context_class` (and that `eqs.set_test_purpose`, `eqs.remove_test` and
`eqs.create overwrite:true` avoid through the file's `SendMutationWithSaveReport` measured save).
The rest of the namespace still carries the old contract, so one namespace answers `saved` two
different ways.

## Expected

Either measure the write (the edit verbs can reuse `SendMutationWithSaveReport`; a fresh create
must weigh the `McpSafeAssetSave` "do not save newly created assets immediately on 5.7+" note) or
report honestly with `AddMarkDirtySaveReport` (`saved` measured, `markedForSave`, `pendingFlush`).

## Found by

Review of `F-eqs-set-test-purpose-and-remove-test` (overwrite path measured; the fresh-create path
and the sibling verbs were left out of scope).

## History
- `#1-filed` `OPEN` developer — Filed while fixing the review findings on `F-eqs-set-test-purpose-and-remove-test`; not fixed here (pre-existing, sibling verbs out of that ticket's scope). `eqs.md` documents that a fresh `eqs.create` still echoes `saved`.
