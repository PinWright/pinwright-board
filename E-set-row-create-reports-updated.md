---
id: E-set-row-create-reports-updated
title: "data_table.set_row(createIfMissing=true) reports result key \"updated\" when it actually adds a new row"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [data-table, set-row, create-if-missing, result-misreport, added-vs-updated]
---

# `data_table.set_row` with `createIfMissing=true` mislabels a created row as `updated`

`data_table.set_row` returns a result object whose **key** names the operation
performed: `{"updated":"<rowName>","row":{...}}` when it overwrites an existing
row. Its sibling `data_table.add_row` reports the same kind of write as
`{"added":"<rowName>","row":{...}}`. So the response key is the natural,
machine-readable signal for "did this create a new row or modify an existing
one" — they deliberately differ (`added` vs `updated`).

But when `set_row` is called with `createIfMissing=true` on a row that does
**not** exist, it correctly creates the row (the wiki: *"When
createIfMissing=true, falls back to add-row semantics if the row does not
exist."*) — yet the result still comes back keyed `"updated"`, not `"added"`.
The response misreports a create as an update. A caller that distinguishes
create-vs-update by reading the result key (the only thing in the payload that
carries that distinction) is told the row pre-existed when in fact it was just
born.

The asymmetry is the giveaway: the exact same row-creation operation reports
`added` through `add_row` but `updated` through
`set_row(createIfMissing=true)`. The wiki for `set_row` itself describes this
branch as "add-row semantics", so the documented intent is an add — only the
result label disagrees with both the docs and the sibling verb.

## Verbatim repro (live, replay-confirmed via `mcp__editor-automation__call`)

Asset: `/Game/Global/DemoRoom/Misc/DT_Colourways.DT_Colourways` (rowStruct
`S_Colour`, single FColor field `colour`).

1. `data_table.set_row` `{assetPath:".../DT_Colourways.DT_Colourways", rowName:"ZZ_ReplayProbe_NonExistent", values:{colour:{r:1,g:2,b:3,a:4}}, createIfMissing:true}`
   (row did NOT exist beforehand) →
   **`{"updated":"ZZ_ReplayProbe_NonExistent","row":{"colour":{"b":3,"g":2,"r":1,"a":4}}}`**
   — the row was newly **created**, but the result is keyed `"updated"`.
2. Contrast — `data_table.add_row` `{..., rowName:"ZZ_ReplayProbe_AddRow", values:{colour:{r:9,g:10,b:11,a:12}}}` →
   `{"added":"ZZ_ReplayProbe_AddRow","row":{...}}` — same create operation, keyed `"added"`.
3. Contrast — `data_table.set_row` `{..., rowName:"ZZ_ReplayProbe_NonExistent", values:{...}, createIfMissing:false}` on the (now-existing) row →
   `{"updated":"ZZ_ReplayProbe_NonExistent","row":{...}}` — a genuine update, also keyed `"updated"`, indistinguishable from step 1.

So `{"updated": ...}` returned from a `createIfMissing=true` call that created
the row (step 1) — identical to the key returned for a real update (step 3) —
is the quotable misreport. (Probe rows removed after the replay; table left in
its prior state.)

**Impact:** an agent doing idempotent upsert via
`set_row(createIfMissing=true)` cannot tell from the response whether it created
a new row or overwrote an existing one — the two cases are byte-identical
(`"updated"`). To get a reliable "was this a create?" signal it must
`list_rows`/`describe` first, defeating the point of the `createIfMissing`
convenience flag. Medium severity: the write itself is correct and the row
lands, but the create-vs-update report — the only signal in the payload — is
wrong, so the upsert caller trusts a fact the handler misreports.

**Workaround:** call `data_table.describe`/`list_rows` to check row existence
before the upsert if you need to know create-vs-update; or prefer `add_row`
(which reports `added` and errors on duplicates) when you specifically intend a
create.

**Fix:** in the `set_row` handler, when `createIfMissing=true` takes the
add-row fallback path (row did not exist), key the result `"added"` to match
`add_row` and the documented "add-row semantics"; keep `"updated"` only for the
overwrite-existing path. Optionally include a boolean `created` flag so callers
get an unambiguous signal without depending on which key is present.

## History
- `#1-initial-repro` `OPEN` reporter — `data_table.set_row {rowName:<new>, createIfMissing:true}` on a non-existent row creates the row (per wiki "falls back to add-row semantics") but returns `{"updated":<rowName>, row:{...}}` — keyed `updated`, identical to a genuine overwrite. The sibling `data_table.add_row` reports the same create as `{"added":<rowName>, ...}`. Replay-confirmed live against `mcp__editor-automation__call` on `/Game/Global/DemoRoom/Misc/DT_Colourways.DT_Colourways`: create-via-set_row → `updated`; add_row → `added`; update-via-set_row → `updated` (indistinguishable from the create case). Ergonomic result-misreport: the response key is the only create-vs-update signal in the payload and it's wrong for the createIfMissing-creates path. Probe rows cleaned up after replay.
- `#2-retriage` `OPEN` triage — Low→Medium: result key misreports a create as updated (wrong data the upsert caller trusts) but bounded edge branch, recoverable via list_rows.
- `#3-fix` `IN-REVIEW` developer — Fixed the create-vs-update result misreport. In `data_table.set_row`, the `createIfMissing=true` create branch (which had provably just created the row via `ApplyValuesToNewRow`) now keys the result `"added"` (matching the sibling `add_row` and the handler's documented "add-row semantics") plus an explicit `created:true`, instead of the old `"updated"`; the genuine in-place overwrite path keeps `"updated"` and now also carries `created:false`, so callers get an unambiguous create-vs-update signal via both the key and the boolean. Updated the `set_row` registered summary to document the two result shapes. File: `Source/PinWright/Private/Handlers/DataTable/DataTableAuthoringHandler.cpp` (create branch ~:538-545, update branch ~:567-571, summary :477). Regression tests in `Source/PinWright/Private/Tests/DataTable/TestDataTableAuthoring.cpp`: extended `FDataTableSetRowMissingCreateIfMissingTrueAddsTest` (PinWright.data_table.set_row.CreateIfMissing) to assert the create result keys `added`+`created:true` and does NOT key `updated` (fails if the fix is reverted), and `FDataTableSetRowExistingUpdatesBytesTest` (PinWright.data_table.set_row.UpdatesExisting) to assert the overwrite result keys `updated`+`created:false` and does NOT key `added`. Also corrected the stale "Low severity" line in the body to Medium. Did not compile/run tests (later phase).
</content>
</invoke>
