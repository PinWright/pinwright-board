---
id: E-data-table-invalid-row-values-no-field-hint
title: "data_table.add_row/set_row [INVALID_ROW_VALUES] error names the row+struct but not the offending field, the bad value, or (for enums) the valid literals — forces an asset.dump detour"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [data-table, add_row, set_row, error-messages, error-hint, enum, invalid-row-values, discovery]
---

# `[INVALID_ROW_VALUES]` is a dead-end error — no field name, no bad value, no valid-set hint

When `data_table.add_row` / `data_table.set_row` fail to convert a supplied
`values` field (a type mismatch, or an invalid enum literal on a field whose key
*did* match), the handler returns a flat error:

> `[INVALID_ROW_VALUES] Failed to apply values to row '<rowName>' of struct '<StructName>'`

The message names the row and the struct but **not which field** failed, **not
the value** that was rejected, and — for the common enum case — **not the set of
valid literals**. So a caller that passed one bad field among several has no
in-band signal of which one to fix, and must leave the tool surface to
`asset.dump` the field's enum type and read the legal values out of the sidecar.

This is the same dead-end-error ergonomic shape the board already tracks for
`audio.authoring.add_metasound_node` (`E-add-metasound-node-error-no-hint`,
OPEN) and has fixed elsewhere (`E-make-struct-error-hint`,
`E-bpir-createwidget-pin-hint`, both DONE): an accurate error that doesn't name
the offending input or point at the valid set converts a one-shot fix into a
discovery detour.

## Distinct from the silent-drop bug

`B-data-table-row-values-silent-drop` (judge-filed) is about keys that **don't
match a field** being dropped silently with a *success* result. This ticket is
the opposite branch: a key that **did match a field** but whose **value** can't
be converted, which *does* error with `[INVALID_ROW_VALUES]` — but the error is
uninformative. The bug ticket explicitly carves this out as separate ("this is
independent of the existing `[INVALID_ROW_VALUES]` error, which only fires on a
type/enum conversion failure of a key that DID match a field"). Different branch,
different remedy (error-message enrichment, not key validation).

## Process friction (this task)

DataTable migration task (focus `data_table.set_row_struct`): adding a `Studio`
room preset whose `wall Drop` field (an `E_EnclosureLevel` enum) was set to
`"None"`. Friction note (verbatim):

> "the E_EnclosureLevel enum rejected `None` with [INVALID_ROW_VALUES] until I
> dumped the enum to find valid values Closed/Partial/Open. add_row's error
> message didn't say which field/value was bad."

Call-log cost: a failed `add_row` (`"Studio wall Drop=None (invalid enum)"`,
`[INVALID_ROW_VALUES]`) → an out-of-band `asset.dump` on `E_EnclosureLevel`
(`"find valid enum values"`) → a corrected `add_row` (`"Studio wall Drop=Open
(valid)"`). The enum-dump detour exists only because the error withheld both the
field name and the valid literals.

## What it should do / how to fix

Enrich the `[INVALID_ROW_VALUES]` error so it carries the failure detail the
caller needs to self-correct without leaving the tool:

> `[INVALID_ROW_VALUES] Field 'wall Drop' of struct 'S_RoomSettings': value
> "None" is not a valid E_EnclosureLevel. Valid values: Closed, Partial, Open.`

Cheapest useful form: when the per-field JSON→struct apply fails, capture the
failing `FProperty` (field display key) and the raw value, and — when it is an
enum property (`FByteProperty`/`FEnumProperty`) — enumerate the `UEnum`'s entries
for the valid set. Even just the **field name + the rejected value** (without the
enum enumeration) would remove the "which field?" half of the detour. This pairs
naturally with the per-field validation the silent-drop bug proposes — both want
the handler to identify the specific failing field rather than reporting at the
whole-row granularity.

**Workaround:** on `[INVALID_ROW_VALUES]`, `asset.dump` the row struct
(`user_defined_struct.json`) to find each field's type, then `asset.dump` any
enum-typed field's `UEnum` to read its legal literals, and retry with a value
from that set.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of a DataTable migration task (focus `data_table.set_row_struct`). `data_table.add_row` returned `[INVALID_ROW_VALUES] Failed to apply values to row 'Studio' of struct 'S_RoomSettings'` for `wall Drop="None"` (an `E_EnclosureLevel` enum) — naming the row + struct but not the offending field, the rejected value, or the valid literals. Recovery required an out-of-band `asset.dump` on `E_EnclosureLevel` to find Closed/Partial/Open, then a corrected `add_row`. Same dead-end-error shape as `E-add-metasound-node-error-no-hint` (OPEN) and the DONE `E-make-struct-error-hint` / `E-bpir-createwidget-pin-hint`. Distinct from the judge-filed `B-data-table-row-values-silent-drop`: that is the unmatched-key *silent drop* (success result); this is the matched-key value-conversion *error* branch, which the bug ticket itself carves out as separate. Proposed fix: enrich `[INVALID_ROW_VALUES]` to name the failing field + bad value and (for enum properties) enumerate the valid literals from the `UEnum`. Dedup: ripgrep across OPEN/closed found no ticket on the `data_table` `INVALID_ROW_VALUES` message; the only references are inside the silent-drop bug (carving this out) and unrelated namespaces.
- `#2-enrich-invalid-row-values` `IN-REVIEW` developer — Enriched the flat `[INVALID_ROW_VALUES]` error at both sites in `DataTableAuthoringHandler.cpp` (the `ApplyValuesToNewRow` new-row/createIfMissing path and the in-place `set_row` update). Added exported helper `DataTableAuthoringInternal::DescribeRowValueFailure(Struct, Values)` (new `Private/Handlers/DataTable/DataTableAuthoringInternal.h`, implemented in the handler `.cpp`): when the stock `FJsonObjectConverter::JsonObjectToUStruct` returns its bare bool, it walks the row struct's authored fields against the supplied `values` and, for the enum-literal case that escapes `ValidateValuesAgainstStruct` (an `FByteProperty`-with-enum or `FEnumProperty` given an unresolvable string), returns a detail naming the field, the rejected value, the enum, and the valid literals — e.g. `Field 'WallDrop' of struct 'S_RoomSettings': value "None" is not a valid ETestEnclosureLevel. Valid values: Closed, Partial, Open`. Enum resolution mirrors `Utils/PropertyImport.cpp` (`GetValueByNameString` + `GenerateFullEnumName`); literal enumeration mirrors `Utils/PropertyExport.cpp`'s `UEnum` walk, skipping the synthetic `_MAX`/Hidden sentinels. The detail is appended to the original whole-row message (empty when no specific field can be pinpointed, so the legacy message stands). Regression test `EditorAutomationRpcGateway.data_table.row_value_failure.EnumHint` (`Private/Tests/DataTable/TestDataTableRowValueFailure.cpp` + fixture `TestDataTableRowValueFailureFixture.h` with an `enum class:uint8`/FEnumProperty + a `TEnumAsByte`/FByteProperty row struct) exercises the production symbol directly: asserts the bad `enum class` literal yields a detail naming field+value+enum+all three literals and omitting `_MAX`, the bad `TEnumAsByte` literal is pinpointed too, and valid literals yield an empty (no-failure) detail. Reverting the enrichment makes the helper return empty and the field/value/literal assertions fail.
