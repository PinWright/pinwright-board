---
id: B-data-table-enum-values-reach-converter
title: "data_table.add_row / set_row pass an unresolvable enum literal to the engine JSON converter, which logs two LogJson Error lines per call"
status: IN-REVIEW
severity: Low
category: bug
tags: [data-table, log-noise, validation, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T16:53:50Z
---

# Unresolvable enum literals reach `FJsonObjectConverter`

`data_table.add_row`, `data_table.set_row` and `set_row`'s `createIfMissing` branch
(`Source/PinWright/Private/Handlers/DataTable/DataTableAuthoringHandler.cpp`) already build a
typed `INVALID_ROW_VALUES` error naming the field, the rejected value and the valid literals
(`DataTableAuthoringInternal::DescribeRowValueFailure`, `E-data-table-invalid-row-values-no-field-hint`).
They only did so *after* `FJsonObjectConverter::JsonObjectToUStruct` had refused the value. The
engine logs that refusal on every call:

```
LogJson: Error: JsonValueToUProperty - Unable to import enum ETestDataTableAtomicMode from string value Bogus for property Mode
LogJson: Error: JsonObjectToUStruct - Unable to import JSON value into property Mode
```

So a caller's typo in an enum field put two engine `Error:` lines in the editor log, which read like an
engine fault, and failed `PinWright.data_table.set_row.AtomicOnLateConversionFailure` once engine errors
stopped being masked (`B-suppress-log-errors-static-leaks`; run log
`Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log`).

**Fix:** run the existing enum-literal check before the converter on all three paths and return the same
`INVALID_ROW_VALUES` message; the converter is only reached with resolvable literals. The check's
resolver now tries the converter's own predicate first
(`GetValueByName(FName, EGetByNameFlags::CheckAuthoredName)`, present from UE 5.3), so a value the
engine accepts (for example a user-defined enum's authored name) is never refused early.

**Scope:** the pre-check walks the payload the way the converter imports it: top-level fields, array
and set elements, map keys and values, nested struct fields and fixed-size C arrays, and the error
names the full property path (`Modes[1]`, `Inner.Mode`, `Lookup["Key"]`). What it deliberately leaves to
the converter is content only the converter's own parser can judge (a malformed date, an unresolvable
object path); for those the error now carries the converter's `OutFailReason` (e.g. `Unable to import
JSON value into property When: Unable to import JSON string into DateTime property When`) instead of the
bare "Failed to apply values" line, and the engine still logs its two LogJson lines.

## History
- `#1-enum-typo-logs-engine-errors` `OPEN` reporter — Unmasked full suite: `data_table.set_row.AtomicOnLateConversionFailure` failed on the two LogJson errors its deliberate `Mode: "Bogus"` input provokes. The verb already knew how to describe the failure and only used that after the engine had logged.
- `#2-precheck-enum-literals` `IN-REVIEW` developer — Added `DescribeUnresolvableEnumValues` in `DataTableAuthoringHandler.cpp`, called before `FDataTableEditorUtils::AddRow` in `ApplyValuesToNewRow` and before the scratch conversion in `set_row`. Both return `INVALID_ROW_VALUES` with the unchanged message shape. `EnumStringResolves` now checks the converter's `CheckAuthoredName` lookup first. Regression check: `AtomicOnLateConversionFailure` fails on any LogJson error if the pre-check regresses; its code/message/row-unchanged assertions are unchanged. Compile-checked only; needs a suite run.
- `#3-nested-walk-and-late-coverage` `IN-REVIEW` developer — Closed both limits of `#2`. (1) `DescribeRowValueFailure` now recurses through arrays, sets, map keys and values, nested structs and fixed-size arrays with full property paths; new tests `PinWright.data_table.set_row.EnumArrayElementRejectedBeforeConverter` (`Modes[1]`, row unchanged) and `PinWright.data_table.add_row.NestedStructEnumRejectedBeforeConverter` (`Inner.Mode`, no orphan row) declare no engine errors, so a LogJson line fails them. (2) `TestDataTableAtomicityFixture.h` gains `Modes`, `Inner` and a last field `FDateTime When`; `AtomicOnLateConversionFailure` now sends valid `Label`/`Score`/`Mode` plus `When: "not-a-date"`, which only the converter rejects after writing the earlier fields to the scratch copy, declares its two LogJson errors with count 1 each, and keeps the byte-identical/field rollback assertions. The converter's `OutFailReason` is now appended when the pre-check has nothing to say, so that error names `When`. Compile-checked only (`-SingleFile` on the handler and three data_table test files); `check_test_ids` clean; needs a suite run.
