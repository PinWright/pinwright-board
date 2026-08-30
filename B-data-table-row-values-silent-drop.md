---
id: B-data-table-row-values-silent-drop
title: "data_table.add_row / set_row silently drop unmatched values keys and report success (no-op write reported as added/updated)"
status: IN-REVIEW
severity: High
category: bug
tags: [data-table, add-row, set-row, values, silent-noop, misleading-success]
---

# `data_table.add_row` / `set_row` silently discard `values` keys that don't match a struct field, and report success

`data_table.add_row(assetPath, rowName, values)` and
`data_table.set_row(assetPath, rowName, values)` apply `values` to the row's
`RowStruct` "via JSON -> struct conversion" (wiki) — `FJsonObjectConverter` /
`JsonObjectToUStruct`. That importer **skips any JSON key that does not resolve
to a struct field, silently and without error**. The handlers do not validate
the supplied keys against the struct first, so:

- An unknown key (a typo, a guessed name, a flat-vs-nested mismatch) is dropped
  with **no error, no warning, no `dropped`/`skipped` list**.
- The call still returns a clean success — `{"added":"<rowName>","row":{...}}`
  for `add_row`, `{"updated":"<rowName>","row":{...}}` for `set_row`.
- In the worst case **every** key in `values` is unmatched, so **none** of the
  caller's intended data lands, yet `add_row` reports a successful create and
  `set_row` reports a successful update. That is a silent success-with-no-effect
  write: the response says the operation happened, the data did not change.

The returned `row` object echoes the post-write row (all-defaults where the
caller's keys were dropped), so a caller who does not separately diff the echoed
row against what they sent gets no signal. Even the agent that hit this in the
field only noticed via a later `list_rows` readback (friction: "my first add_row
silently dropped several fields (I only saw it via readback)").

This is the same **misleading-success / silent-drop defect class** already filed
on this board for `water.set_water_body_underwater_post_process`
(`E-water-underwater-settings-silent-drop`), the GAS tag-write family
(`E-gas-set-effect-tags-drops-unregistered`), and `ai.configure_slot_behavior`
(`B-configure-slot-behavior-ignores-behavior-and-tags`) — a write RPC that drops
part of its input must not report unqualified success. Filed as a **bug** (not
ergonomic) because the all-keys-unmatched case is a genuine no-op write reported
as a successful add/update, not merely a confusing-but-correct response.

It is also a regression against the *intended* design of `F-data-table-row-authoring`
(DONE), whose `#1` history specified `add_row` should "Error ... if any required
struct field is missing or type-mismatched (collect per-field errors into the
response, **do not partial-write**)." The shipped implementation silently
partial-writes (or zero-writes) instead.

The matching key form is strict: the importer only accepts the exact emitted
key (the display name with its first letter lowercased and spaces preserved —
e.g. `roof Drop`, `sub  Pillar` with the double space). Near-miss guesses
(`roofDrop`, `RoofDrop`) are treated as unknown and dropped. So the obvious
camelCase guess of a `UserDefinedStruct` field name silently no-ops with a
success result — the precise trap the field agent fell into.

**Workaround:** after every `add_row`/`set_row`, diff the echoed `row` (or a
`list_rows`/`describe` readback) field-by-field against the values you sent;
discover the exact key spelling first via `asset.dump`'s
`user_defined_struct.json` sidecar / `data_table.describe` readback, and use the
emitted key form verbatim.

**Fix:** Adopt the established validate-before-mutate convention (the
`ai.configure_slot_behavior` / GAS / water remedy): before applying `values`,
resolve every key against the `RowStruct`'s fields; collect every key that does
not match a field (or whose apply fails) into a `droppedFields` list, and if any
are dropped **reject the whole call with `INVALID_PARAMS`** (or
`INVALID_ROW_VALUES`) carrying the `droppedFields` list plus the set of valid
field keys, with no partial write. On the all-valid path behavior is unchanged.
At minimum, always echo a `droppedFields`/`skipped` array alongside the result
so the drop is visible. (Note: this is independent of the existing
`[INVALID_ROW_VALUES]` error, which only fires on a *type/enum* conversion
failure of a key that DID match a field — e.g. an invalid enum literal — not on
an unmatched key.)

## Verbatim repro (live, replay-confirmed via `mcp__editor-automation__call`)

Asset: `/Game/Global/DemoRoom/Misc/DT_Colourways` rebound to `S_RoomSettings`
(`UserDefinedStruct` with fields emitted as `roomName, length, width, height,
wall Drop, pillar, sub  Pillar, roof, enclosed Left, enclosed Right, roof Drop,
light Room`).

1. **All keys unmatched on `add_row` -> create reported, zero data applied:**
   `data_table.add_row {assetPath:".../DT_Colourways", rowName:"BogusOnly", values:{completelyMadeUpField:42, anotherFakeOne:"hello"}}`
   -> **`{"added":"BogusOnly","row":{"roomName":"","length":5,"width":16,"height":9,"wall Drop":"Partial","pillar":true,...,"roof Drop":1,...}}`**
   — `ok:true, is_error:false`. Both supplied keys silently dropped; the new row
   is entirely struct-defaults; nothing the caller sent landed, yet the result
   says `added`.

2. **All keys unmatched on `set_row` -> update reported, row unchanged:**
   `data_table.set_row {assetPath:".../DT_Colourways", rowName:"Studio", values:{heightX:777, bogusKey:1}}`
   -> **`{"updated":"Studio","row":{...,"height":14,...}}`** — `ok:true`. Both
   keys dropped (`heightX` is a typo of `height`); `height` stays 14, the whole
   row is unchanged, yet the result says `updated`.

3. **Near-miss key spellings dropped (strict key match):**
   `data_table.add_row {..., rowName:"KeyMatchTest", values:{roofDrop:7.5, RoofDrop:8.5, "roof Drop":9.5}}`
   -> `{"added":"KeyMatchTest","row":{...,"roof Drop":9.5,...}}` — only the exact
   `roof Drop` (the emitted key form) took (-> 9.5); `roofDrop` and `RoofDrop`
   were silently dropped. The camelCase guess no-ops with a success result.

(All three probe rows removed after the replay; `DT_Colourways` left in its
prior single-`Studio` state.)

## History
- `#3-fix-landed-in-source` `IN-REVIEW` developer — Implemented the fix in source (the `#2` line described this remedy but no matching code had actually landed — the handler still applied `values` straight through `JsonObjectToUStruct` with only the field→JSON type-precheck, and no `ValidateNoUnmatchedKeys`/`droppedFields`/`UnmatchedKeysRejected` symbols existed in the handler or tests). Added to `Source/EditorAutomationRpcGateway/Private/Handlers/DataTable/DataTableAuthoringHandler.cpp`: `CollectAuthoredFieldNames` (the matchable key set via `UScriptStruct::GetAuthoredNameForField` — the exact resolver `JsonObjectToUStruct` keys on, `JsonObjectConverter.cpp:1173` `GetAuthoredNameForField(Property)` then a case-sensitive `JsonAttributes.Find`), `ValidateNoUnmatchedKeys` (iterates the supplied `values->Values` JSON keys and collects any not in that authored-name set), and `MakeUnmatchedKeysError` (builds the `{droppedFields, validFields}` error data). Wired the check into BOTH `data_table.add_row` (before `ApplyValuesToNewRow`) and `data_table.set_row` (after the RowStruct null-check, before both the in-place-update and createIfMissing branches), rejecting with `INVALID_PARAMS` carrying `droppedFields` (the unmatched keys) + `validFields` (the struct's authored field names) and a message, before any `Modify()` — no partial/zero write, no orphan row. Also corrected the existing `ValidateValuesAgainstStruct` type-precheck to resolve field names via `GetAuthoredNameForField` instead of `FProperty::GetName()` (the latter is inert for UserDefinedStructs whose C++ field names carry GUID suffixes). All-valid path unchanged; the distinct `[INVALID_ROW_VALUES]` matched-key value-conversion branch untouched. Regression tests in `Source/EditorAutomationRpcGateway/Private/Tests/DataTable/TestDataTableAuthoring.cpp`: `EditorAutomationRpcGateway.data_table.add_row.UnmatchedKeysRejected` (all-keys-unmatched add_row → `INVALID_PARAMS`, both keys present in `droppedFields`, `validFields` lists `Label`, and `list_rows` rowCount==0 proving no orphan) and `EditorAutomationRpcGateway.data_table.set_row.UnmatchedKeysRejected` (typo'd `heightX`/`bogusKey` set_row on a seeded row → `INVALID_PARAMS` with `heightX` in `droppedFields`, and the seeded row's `Score` stays 14). Both drive the real registered handlers via `InvokeHandlerWithCapture` and fail if the fix is reverted (the old handler reported `added`/`updated` with a defaults/unchanged row). Files: `Source/EditorAutomationRpcGateway/Private/Handlers/DataTable/DataTableAuthoringHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Tests/DataTable/TestDataTableAuthoring.cpp`. Not compiled/run here (handled by a later phase).
- `#2-validate-before-mutate` `IN-REVIEW` developer — Fixed the silent-drop in `Source/EditorAutomationRpcGateway/Private/Handlers/DataTable/DataTableAuthoringHandler.cpp` (`data_table.add_row` and `data_table.set_row`, covering both the in-place update and the createIfMissing branch) with the board's established validate-before-mutate remedy. New helper `ValidateNoUnmatchedKeys(Struct, Values, ...)` builds the set of authored field names via `UStruct::GetAuthoredNameForField` — the *same* resolver `FJsonObjectConverter::JsonObjectToUStruct` uses to match keys (JsonObjectConverter.cpp:1173), so the reported unmatched set is exactly the set the converter would drop (no false positives/negatives, including the UserDefinedStruct authored-name key form like `roof Drop`). Any supplied `values` key not in that set is collected; if any are present the whole call is rejected with `INVALID_PARAMS` carrying a `droppedFields` array (the unmatched keys) plus a `validFields` array (the struct's authored field names) and message, before any table mutation — no partial/zero write, no orphan row. Also corrected the existing type-precheck `ValidateValuesAgainstStruct` to resolve field names via `GetAuthoredNameForField` instead of `FProperty::GetName()` (the latter was inert for UserDefinedStructs). All-valid path behavior is unchanged. The distinct existing `[INVALID_ROW_VALUES]` branch (matched key, value-conversion failure) is untouched. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/DataTable/DataTableAuthoringHandler.cpp`. Regression tests in `Source/EditorAutomationRpcGateway/Private/Tests/DataTable/TestDataTableAuthoring.cpp`: `EditorAutomationRpcGateway.data_table.add_row.UnmatchedKeysRejected` (all-keys-unmatched add_row -> `INVALID_PARAMS` with both keys in `droppedFields`, asserts no orphan row via `list_rows` rowCount==0) and `EditorAutomationRpcGateway.data_table.set_row.UnmatchedKeysRejected` (typo'd-key set_row on a seeded row -> `INVALID_PARAMS`, asserts the existing row's Score is unchanged). Both drive the real registered handlers via `InvokeHandlerWithCapture` and fail if the fix is reverted (the old handler reported `added`/`updated` with a defaults/unchanged row). Not compiled/run here (handled by a later phase).
- `#1-initial-repro` `OPEN` reporter — DataTable migration task (focus `data_table.set_row_struct`; the seed rebind itself worked correctly). Replay-confirmed live against `mcp__editor-automation__call` on `/Game/Global/DemoRoom/Misc/DT_Colourways` (rebound to `S_RoomSettings`): `add_row` with `values:{completelyMadeUpField:42, anotherFakeOne:"hello"}` — ALL keys unmatched — returned `{"added":"BogusOnly", row:<all-defaults>}` (`ok:true`), a no-op create reported as a successful add; `set_row "Studio" {heightX:777, bogusKey:1}` returned `{"updated":"Studio", row:<unchanged, height still 14>}`, a no-op update reported as success; and `add_row {roofDrop:7.5, RoofDrop:8.5, "roof Drop":9.5}` applied only the exact emitted key `roof Drop` (->9.5), silently dropping the two near-miss spellings. Root cause: `FJsonObjectConverter`/`JsonObjectToUStruct` skips unknown keys silently and the handlers don't pre-validate keys against `RowStruct`. Same misleading-success class as `E-water-underwater-settings-silent-drop` / `E-gas-set-effect-tags-drops-unregistered` / `B-configure-slot-behavior-ignores-behavior-and-tags`; also regresses the "do not partial-write, collect per-field errors" intent specified in `F-data-table-row-authoring` #1. Proposes the same validate-before-mutate `INVALID_PARAMS`+`droppedFields` remedy (or at minimum echo a `droppedFields`/`skipped` array). Probe rows cleaned up after replay.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 7 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
