---
id: F-ftext-string-table-loctable
title: "FText string-table support — accept LOCTABLE(...) in widget XML and BPIR, decompile back to LOCTABLE form"
status: DONE
severity: Medium
category: feature
tags: [bpir, widget-xml, ftext, localization, string-table]
---

# FText string-table (LOCTABLE) support

Today the MCP can author FText with `NSLOCTEXT("ns", "key", "source")`
in both widget XML attributes and BPIR pin defaults — that path goes
through `PropertyUtils::CoerceStringToPersistedFText` →
`FTextStringHelper::CreateFromBuffer`, which understands NSLOCTEXT
natively. NSLOCTEXT gives FText a namespace+key identity and is enough
for the gather pipeline to pick the string up at cook time.

What does **not** work today is referencing an existing `UStringTable`
asset, i.e. producing an FText with `FTextHistory_StringTableEntry` so
that at runtime `FText::FromStringTable(TableId, Key)` resolves the
display string from the table. There is no MCP-authorable syntax for
this — neither `LOCTABLE("Table", "Key")` nor any structured
`{ table, key }` form is recognized, and the BPIR decompiler never
emits one (it only emits `NSLOCTEXT(...)` or bare invariant strings).

For PDS, this means any widget BP / BPIR-authored UMG that should
resolve from a shared string-table asset has to be edited by hand in
the editor — the MCP-authored versions silently downgrade to
NSLOCTEXT-keyed inline literals.

## Scope

1. **Widget XML import** (`widget.import_xml` family): accept
   `LOCTABLE("TableId", "Key")` as an FText attribute value, produce an
   FText backed by `FTextHistory_StringTableEntry`.
2. **BPIR compile**: same syntax in BPIR source for any FText pin
   default; routes through the same coercion entry point.
3. **BPIR decompile**: when an existing FText pin default has a
   `FTextHistory_StringTableEntry`, emit `LOCTABLE("TableId", "Key")`
   so round-trip preserves the string-table linkage. Today such
   defaults round-trip to the *resolved* display string and detach
   from the table.
4. **Documentation**: extend `bpir-language-reference.md` to list the
   three FText literal forms — `"plain"` (invariant),
   `NSLOCTEXT(...)`, `LOCTABLE(...)`.

## Why it matters

PDS uses translation tables for Russian + English. Widgets and BP
defaults that are authored against a `UStringTable` asset are the
strongest localization wiring (one source of truth, hot-reload
friendly). Without LOCTABLE support in the MCP, any decompile-edit-
recompile pass strips that linkage — and any green-field
MCP-authored widget can't opt into the table without manual editor
fixup.

This is the LOCTABLE counterpart to the NSLOCTEXT round-trip fix
already in IN-REVIEW under `B-bpir-ftext-localization-metadata-lost`.
That ticket fixes the NSLOCTEXT path; this one adds the third form.

## Proposed implementation

UE 5.6's `FTextStringHelper::CreateFromBuffer` already parses
`LOCTABLE("...","...")` natively (engine
`Runtime/Core/Private/Internationalization/TextHistory.cpp`,
`FTextHistory_StringTableEntry::ReadFromBuffer`), and the same engine
path emits `LOCTABLE(...)` from `FText::ToString()` via
`WriteToBuffer`. So the original three-point compile-side proposal
(PropertyUtils parser + BpirValueResolver literal rule + new
construction logic) was overscoped. Two narrow gaps remain:

1. **`Utils/PropertyUtils.cpp` — `CoerceStringToPersistedFText`:**
   after `FTextStringHelper::CreateFromBuffer` succeeds, the existing
   acceptance gate requires `HasLocalizationIdentity(ParsedText)`
   (i.e. non-empty `FTextInspector::GetNamespace`/`GetKey`). String-
   table-backed FTexts carry identity via `FTextHistory_StringTableEntry`,
   not via Namespace/Key, so that gate rejects them. Added a second
   acceptance branch on `ParsedText.IsFromStringTable()`. One branch,
   no new helper, no new include.

2. **`Decompiler/BpirDecompiler.cpp` — `ResolveInputValue`'s text-default
   block:** added a first branch that, when
   `InputPin->DefaultTextValue.IsFromStringTable()` is true, calls
   `FTextInspector::GetTableIdAndKey` and emits
   `LOCTABLE("<TableId>", "<Key>")` via
   `BpirStructLiteralUtils::EscapeBpirStringInner` plus the standard
   `EscapeBpirString` outer wrap. Reuses the same escape helpers the
   NSLOCTEXT branch already uses. Falls through to the existing
   NSLOCTEXT and bare-display branches when `GetTableIdAndKey` fails.

`BpirValueResolver` and `CodePinResolver` need no changes — their
existing quoted-string literal path carries `LOCTABLE(...)` through
verbatim once PropertyUtils accepts it.

3. **`docs/bpir-language-reference.md`:** documented the third FText
   literal form alongside the existing NSLOCTEXT and bare-string
   descriptions; noted the table asset must be loadable at runtime.

4. **`Source/PinWright/Private/Tests/Bpir/TestBpirFTextStringTableRoundTrip.cpp`:**
   new `FFTextStringTableRoundTripTest`. Registers a transient string
   table; compile-side asserts `CoerceStringToPersistedFText` accepts
   `LOCTABLE(...)` and produces an FText with `IsFromStringTable() == true`
   matching `TableId`/`Key`; decompile-side stamps a
   `FText::FromStringTable(...)` onto a `UK2Node_CallFunction` text pin,
   runs `FBpirDecompiler::Decompile()`, asserts the BPIR output contains
   the LOCTABLE wire form; round-trip feeds the LOCTABLE literal into
   `FCodePinResolver::SetPinDefaultValue` and asserts the recovered FText
   is still string-table-backed with the same `TableId`/`Key`. All three
   scenarios call production code only.

## Acceptance

- Widget XML attribute `Text='LOCTABLE("UI/Common", "OK")'`
  imports as FText resolving from the `UI/Common` string table at
  runtime (display = "OK" if the table has that key in current
  culture).
- BPIR source `Text = LOCTABLE("UI/Common", "OK")` on a `Set Text`
  node compiles to a pin default whose FText has
  `FTextHistory_StringTableEntry { Table = "UI/Common", Key = "OK" }`.
- Decompile of the same BP emits the same `LOCTABLE("UI/Common", "OK")`
  literal — round-trip stable.
- Existing NSLOCTEXT round-trip behaviour unchanged.
- `bpir-language-reference.md` documents all three FText literal
  forms.

## History
- `#1-initial-spec` `OPEN` reporter — Research showed widget XML and BPIR currently support inline FText only as `NSLOCTEXT(...)` or invariant quoted strings; both go through `PropertyUtils::CoerceStringToPersistedFText` which delegates to `FTextStringHelper::CreateFromBuffer`. No syntax accepts a `UStringTable` reference (`LOCTABLE("Table", "Key")` / `FText::FromStringTable`), and the BPIR decompiler never emits a string-table form, so any `FTextHistory_StringTableEntry` default round-trips down to its resolved display string and detaches from the table. Proposes adding LOCTABLE recognition at three points (PropertyUtils coercion, BpirValueResolver literal detection, BpirDecompiler emit) plus a round-trip regression test. ~100 lines, no new architecture.
- `#2-loctable-implemented` `IN-REVIEW` developer — Implemented per Phase 2 simplified scope: extended `CoerceStringToPersistedFText` (Utils/PropertyUtils.cpp) to accept FText parses where `IsFromStringTable()` holds even when namespace+key are empty (table TextId not yet claimed by FTextLocalizationManager); added LOCTABLE emit branch in `BpirDecompiler::ResolveInputValue`'s text-default block reusing `BpirStructLiteralUtils::EscapeBpirStringInner`/`EscapeBpirString` (same path the NSLOCTEXT round-trip uses); documented the third FText literal form in bpir-language-reference.md; added `FFTextStringTableRoundTripTest` covering compile, decompile, and round-trip via production code paths. Engine pre-existing `FTextStringHelper::CreateFromBuffer` LOCTABLE parsing made the originally proposed BpirValueResolver/CodePinResolver changes unnecessary.
- `#3-regression-test-missing` `OPEN` tester — Returned: `system.run_tests tests:["EditorAutomationRpcGateway.bpir.FTextStringTableRoundTrip"]` returned `NO_TESTS_MATCHED` with `missingTests:["EditorAutomationRpcGateway.bpir.FTextStringTableRoundTrip"]`, so the claimed BPIR compile/decompile/round-trip regression is not available in the running editor; widget XML was partially verified by creating `/Game/App/UI/Test/W_McpVerifyTemp_F_ftext_string_table_loctable`, importing `Text="LOCTABLE(&quot;McpVerify/Table&quot;, &quot;OK&quot;)"`, exporting the same LOCTABLE form, and deleting the temp asset.
- `#4-correct-test-name` `IN-REVIEW` developer — Static follow-up: current source already contains the regression in `Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestBpirFTextStringTableRoundTrip.cpp` as exact automation test `EditorAutomationRpcGateway.bpir.round_trip.FTextStringTable`; the returned run used non-existent exact name `EditorAutomationRpcGateway.bpir.FTextStringTableRoundTrip`. No production code change is planned for this ticket; rerun the canonical exact test plus the already documented widget XML verification.
- `#5-verify-fix` `DONE` tester — Verified: `system.run_tests tests:["EditorAutomationRpcGateway.bpir.round_trip.FTextStringTable"]` resolved the test (resolvedTests matches, missingTests empty) and completed with `has_errors:false`; widget XML round-trip on temp `/Game/App/UI/Test/W_McpVerifyTemp_F_ftext_string_table_loctable` imported `Text="LOCTABLE(&quot;McpVerify/Table&quot;, &quot;OK&quot;)"` and exported the same LOCTABLE form back unchanged. Temp asset deleted.
- `#6-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
