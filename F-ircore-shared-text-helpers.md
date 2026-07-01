---
id: F-ircore-shared-text-helpers
title: "IrCore: extract shared text helpers used by MGIR and AGIR"
status: DONE
severity: Low
category: feature
tags: [ircore, mgir, agir, refactor, deduplication]
---

# Extract MGIR/AGIR shared text helpers into IrCore

MGIR (`Private/MGIR/`) and AGIR (`Private/AGIR/`) currently maintain byte-for-byte duplicate copies of several low-level text-processing helpers — escape, quote, position-suffix, comment-strip, delimiter scanning. With AGIR being the second consumer, the 2+-use trigger for promotion is met. A future third IR (Niagara, RigVM, BehaviorTree, etc.) would copy them a third time; this ticket cuts that off.

## Helpers to extract

Pulled from the AGIR Phase 2 quality-review findings (Wave 4 chunks 4A and 4B). Each line cites the AGIR copy and its MGIR original.

### From parsers
- `IsUnescapedQuote(FStringView, int32 Index)` — `AGIRParser.cpp:17` ↔ `MGIRParser.cpp:14`
- `StripTrailingComment(FString& InOut)` — `AGIRParser.cpp:30` ↔ `MGIRParser.cpp:30`
- `FindTopLevelDelimiterPositions(...)` — `AGIRParser.cpp:55` ↔ `MGIRParser.cpp:51`
- `SmartSplit(FString, TCHAR Delim)` — `AGIRParser.cpp:90` ↔ `MGIRParser.cpp:90`
- `FindMatchingChar(FString, int32 OpenIndex, TCHAR Open, TCHAR Close)` — `AGIRParser.cpp:130` ↔ `MGIRParser.cpp:120`
- `UnwrapQuotedName(FString)` — `AGIRParser.cpp:170` ↔ `MGIRParser.cpp:160` (MGIR also strips backticks; provide an optional flag or per-IR shim).
- `TryExtractPosition(FString& InOut, FVector2D& OutPos)` — `AGIRParser.cpp:200` ↔ `MGIRParser.cpp:181`

### From emitters / decompilers
- `EscapeIRString(FString)` — `AGIRTextEmitter.cpp:36` ↔ `MGIRDecompiler.cpp:88`
- `Quote(FString)` — `AGIRTextEmitter.cpp:60` ↔ `MGIRDecompiler.cpp:130`
- `FormatPositionSuffix(FVector2D)` — `AGIRTextEmitter.cpp:219` ↔ `MGIRDecompiler.cpp:144`

### Reflective property emit (partial)
- `IsSafeReflectedProperty(FProperty*)` — `AGIRTextEmitter.cpp:100` (anim version) ↔ `MGIRDecompiler.cpp:349` (material version). The unsafe-flag set is identical; the type allowlist differs only by what each IR rejects (`FPoseLink` for AGIR, `FExpressionInput`-as-struct for MGIR). Refactor MGIR's helper to take a reject-this-struct predicate and share core.

### Scalar value export (deeper)
- `ExportReflectedPropertyValue(FProperty*, const void* Container, FString& Out)` — `AGIRTextEmitter.cpp:145` (`ExportRuntimeFieldValue`) ↔ `MGIRDecompiler.cpp:244`. Both reduce to `Property->ExportText_InContainer` with type-typed dispatch over Bool/Numeric/Enum/Object/etc. Container is `void*` for both — unifies cleanly.

## Proposed location

New file `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/IrCore/IrTextUtils.{h,cpp}`. Sits next to existing `IrCore/IrTokenizer`, `IrCore/IIrGrammar`, `IrCore/IrTypeSpec`, `IrCore/IrCompileDiagnostic`. No new module dependency needed; everything is core UE.

## Result-struct unification (bonus)

`FAGIRPinResolver::WirePoseInput` returns `bool` + `FString& OutErrorCode`; `FMGIRPinResolver::WireExpressionInput` returns `FMGIRWireResult { ErrorCode, ErrorMessage, IsSuccess() }`. Both shapes fit a generic `FIrWireResult` in `IrCore/`. Optional — only do this if the result-struct churn at the call sites feels worthwhile.

## Approach

1. **Land the parser helpers first.** They're pure functions with no UE-class dependencies and no behaviour change. Add to `IrTextUtils.h`, write thin shims in `MGIRParser.cpp` and `AGIRParser.cpp` (`using AGIRParser::TryExtractPosition = IrCore::TryExtractPosition;`), delete the local copies. Tests should pass unchanged.

2. **Land the emitter helpers second.** Same pattern. Verify by inspection that escape rules / position formatting are byte-identical between MGIR and AGIR before sharing — if a future IR wants different escape semantics, make the shared helper take an escape table.

3. **Reflective-property unification last.** Higher risk (the per-property switch is the long tail). Drive by lifting the unsafe-flags filter first (small, low-risk), then the ExportText dispatch in a second pass.

Each pass is independent and small (<200 lines diff). No need to bundle.

## Why severity Low

The duplication is currently maintainable — three files in lock-step, no observed bug or drift. This is preventative refactoring against the next IR landing. Bump severity if Niagara IR / RigVM IR / BehaviorTree IR work starts before this ticket lands.

**Workaround:** none needed; current duplication compiles and works.

**Fix:** Land this as one coherent refactor: add `IrCore/IrTextUtils`, make backticks the canonical delimiter for name tokens, keep double quotes for string payloads, migrate MGIR/AGIR parser and emitter call sites, share the reflective-property safety filter, and add pure helper regression tests.

## History
- `#1-initial-spec` `OPEN` reporter — Filed during AGIR Phase 3 sprint completion summary. AGIR Wave 4 quality-review found 7 parser helpers, 3 emitter helpers, plus the reflective-property emit pipeline duplicated verbatim from MGIR. Refactor deferred from the AGIR fix pass to avoid colliding with same-file parser/emitter mismatch fixes already in flight. Land as 3 independent PRs (parser → emitter → reflection).
- `#2-shared-ir-text-utils` `IN-REVIEW` developer — Added shared `IrCore/IrTextUtils`, migrated MGIR/AGIR parser scanning and emitter quoting/name-token helpers, made backtick-delimited name tokens canonical while keeping double quotes for string payloads, shared reflective-property safety filtering, and added `FIrTextUtilsNameTokenRoundTripTest` plus `FIrTextUtilsBacktickScanningTest`.
- `#3-verify-fix` `DONE` tester — Verified: `IrCore/IrTextUtils.{h,cpp}` present at expected paths; MGIRParser/AGIRParser/MGIRDecompiler/AGIRTextEmitter all call `FIrTextUtils::` (9/12/7/5 refs respectively, no leftover local copies in the migrated functions). Ran `system.run_tests` filter `EditorAutomationRpcGateway.core.ir_text`; both `core.ir_text.name_token.RoundTrip` and `core.ir_text.scanning.IgnoresBacktickDelimitedNames` returned `Result={Success}`.
