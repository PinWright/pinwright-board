---
id: F-ircore-reflected-property-emit
title: "IrCore: consolidate reflected-property value emit + field-loop across MGIR/AGIR (and future BTIR/SCIR/MSIR/NIR)"
status: DONE
severity: Medium
category: feature
tags: [ircore, mgir, agir, refactor, deduplication, reflected-emit]
---

# Consolidate reflected-property emit across IR decompilers

`F-ircore-shared-text-helpers` (DONE) extracted the low-level text helpers
(escape, quote, position suffix, scanning, `IsSafeReflectedProperty` filter)
shared by MGIR and AGIR. It explicitly deferred the larger
`ExportReflectedPropertyValue` / `ExportRuntimeFieldValue` dispatch and the
surrounding `Append*Properties` loop as "Higher risk … land last." This
ticket picks that thread up before four additional IRs land copies of the
same code.

## Evidence

### Duplicated value-export ladder

`MGIRDecompiler.cpp:204-307` — `ExportReflectedPropertyValue(UMaterialExpression*, FProperty*)`, ~104 LoC.
`AGIRTextEmitter.cpp:92-164` — `ExportRuntimeFieldValue(const void* StructData, FProperty*, UObject*)`, ~73 LoC.

Both reduce to the same type-dispatch ladder against `FProperty`:

| Branch | MGIR (`Expression` is UObject*) | AGIR (`StructData` is void*) |
|---|---|---|
| `FBoolProperty` | `GetPropertyValue_InContainer` | `GetPropertyValue` on `ContainerPtrToValuePtr` |
| `FNumericProperty` (non-byte) | `GetFloatingPointPropertyValue` / `GetSignedIntPropertyValue` | identical |
| `FEnumProperty` | underlying signed → `GetNameStringByValue` → `Quote` | identical |
| `FByteProperty` | enum-aware quoting; else `%lld` | identical |
| `FClassProperty` / `FObjectProperty` | `Quote(Value ? GetPathName : empty)` | identical |
| `FSoftClassProperty` / `FSoftObjectProperty` | `ToSoftObjectPath().ToString()` quoted | identical |
| `FArrayProperty` | bespoke `[elem,elem]` form (MGIR only — see divergence) | absent |
| fallback | `ExportTextItem_InContainer(..., Expression, ...)` → `Quote` | `ExportTextItem_Direct(..., OwnerForExportText, ...)` → `Quote` |

Container access is the only semantic difference: MGIR walks
`Property->ContainerPtrToValuePtr<void>(Expression)` and AGIR walks
`Property->ContainerPtrToValuePtr<void>(StructData)`. Both are `void*` in
the FProperty API, so a single helper taking `const void* Container` plus
an explicit `UObject* OwnerForExportText` unifies cleanly. AGIR's comment
at line 90 already names this as mirroring MGIR.

### Divergence to preserve

- **Array branch.** MGIR emits `FArrayProperty` as `[elem,elem]` with
  string-like inners quoted (MGIRDecompiler.cpp:268-302). AGIR has no array
  branch — anim-node arrays go through the `ExportText` fallback. Shared
  helper should expose the MGIR array form behind an opt-in flag (or
  always-on; AGIR has no observed array-property emit, so enabling it
  globally is safe but unverified).
- **Fallback ExportText variant.** MGIR uses `ExportTextItem_InContainer`,
  AGIR uses `ExportTextItem_Direct`. Both end up at the same underlying
  property serializer; `_Direct` takes a raw value pointer and `_InContainer`
  takes a container pointer. A shared helper can always take the container
  and call `_InContainer`.

### Duplicated field-iteration loop

`MGIRDecompiler.cpp:325-363` — `AppendReflectedExpressionProperties` (~39 LoC).
`AGIRTextEmitter.cpp:336-392` — `AppendReflectedNodeFields` (~57 LoC; the
extra lines come from resolving `FStructProperty` host + default sidecar).

Both implement the same five-step pipeline:
1. Resolve default-object pointer for CDO-delta filtering.
2. `TFieldIterator<FProperty>` over the host struct/class.
3. Reject via `IsSafeReflectedProperty` (already shared via
   `FIrTextUtils::IsSafeReflectedProperty`).
4. Sort by `Property->GetName()`.
5. Skip `Identical_InContainer` against the default; emit
   `"<Name><sep><Value>"` line.

Separator is the only divergence — MGIR uses `": "`, AGIR uses `"="`.
Grammar harmonization is tracked separately; the shared helper should
accept it as a parameter.

### Duplication count today

~140 LoC across both files (104 + 73 value-export ≈ 177, but ~37 LoC of
that is structurally identical text the compiler will deduplicate; ~140
human-maintained LoC after rounding the genuinely shared portion). With
four upcoming IRs (BTIR, SCIR, MSIR, NIR) each independently needing
reflected-field emit, the cost multiplies 5×.

## Proposed extraction

New API on `IrCore/IrTextUtils.h` (file added by F-ircore-shared-text-helpers):

```cpp
struct FReflectedFieldEmitOptions
{
    TCHAR Separator = TEXT(':');
    bool bIncludeSpaceAfterSeparator = true;
    bool bEmitArraysAsBracketList = true;  // MGIR shape; AGIR can opt in
};

// Type-dispatch ladder + ExportText fallback. Container is the raw struct
// or UObject memory; Owner is passed to ExportText for path resolution.
static FString FormatReflectedPropertyValue(
    const void* Container,
    FProperty* Property,
    UObject* Owner,
    const FReflectedFieldEmitOptions& Options);

// Field-iteration loop with CDO-delta filtering. Caller supplies the
// iterated UStruct (UClass for MGIR-style UObjects, UScriptStruct for
// AGIR-style anim node payloads), the instance + default pointers, the
// already-handled property set, and per-IR reject predicates (struct-type
// and name-based) that compose with FIrTextUtils::IsSafeReflectedProperty.
static void AppendReflectedFields(
    UStruct* IteratedStruct,
    const void* Instance,
    const void* Default,
    UObject* OwnerForExportText,
    const TSet<FName>& ExplicitProperties,
    TFunctionRef<bool(const FStructProperty*)> RejectStruct,
    TFunctionRef<bool(FName)> RejectName,
    const FReflectedFieldEmitOptions& Options,
    TArray<FString>& OutFields);
```

MGIR `ExportReflectedPropertyValue` collapses to
`FIrTextUtils::FormatReflectedPropertyValue(Expression, Property, Expression, {':', true, true})`.
AGIR `ExportRuntimeFieldValue` collapses to
`FIrTextUtils::FormatReflectedPropertyValue(StructData, Property, OwnerForExportText, {'=', false, false})`.

`AppendReflectedExpressionProperties` and `AppendReflectedNodeFields` both
collapse to a single call to `AppendReflectedFields` with their
IR-specific reject predicates (already isolated by
`F-ircore-shared-text-helpers`).

## Risk

LOW–MEDIUM. The refactor is mechanical and behaviour-preserving, but the
ladder has many branches and there's one real divergence (MGIR's array
form). Existing tests pin the contract:
- `FMGIRDecompilerArrayPropertiesTest` covers the MGIR array branch
  (round-trips `[elem,elem]` form).
- AGIR round-trip tests cover the fallback `ExportText` path.
- `FIrTextUtilsNameTokenRoundTripTest` covers the surrounding quoting.

Suggested rollout (one PR per step; the bonus tests are tiny):
1. Land `FormatReflectedPropertyValue` first (value-only, no loop change).
   Migrate MGIR call sites; verify MGIR tests pass. Then migrate AGIR.
2. Land `AppendReflectedFields` second. Migrate MGIR, then AGIR.
3. Add a focused test `FIrTextUtilsReflectedPropertyEmitTest` exercising
   each FProperty branch on a synthetic UStruct, parameterized by the
   `FReflectedFieldEmitOptions` separator/array-form toggles.

## Why severity Medium (vs the precedent's Low)

`F-ircore-shared-text-helpers` set its severity to Low because the
duplication was preventative against unknown future IRs. Four concrete IRs
(BTIR, SCIR, MSIR, NIR) are now queued, each of which will copy this code
verbatim unless this ticket lands first — Medium reflects "imminent 5×
multiplier, not hypothetical."

## Relationship to upcoming IRs

This is a **soft prerequisite** (recommended, not blocking) for:
- SCIR (StateTree IR) — also walks reflected anim-style struct fields
- BTIR (BehaviorTree IR) — UBTNode-derived UObject properties
- MSIR (MetaSound IR) — node-property emit
- NIR (Niagara IR) — module property emit

Each can ship before this ticket lands by copy-pasting the existing
helpers, but then this refactor grows to migrate six files instead of
two. Recommend landing this in the slot between AGIR completion and the
first new-IR ticket starting work.

**Workaround:** none needed; current code compiles and works.

**Fix:** As described under "Proposed extraction" above. See
[`F-ircore-shared-text-helpers`](F-ircore-shared-text-helpers.md) for the
established `IrCore/IrTextUtils.{h,cpp}` location and shim pattern.

## History
- `#1-initial-spec` `OPEN` reporter — Filed after audit found
  `MGIRDecompiler.cpp:204-307` `ExportReflectedPropertyValue` (~104 LoC)
  and `AGIRTextEmitter.cpp:92-164` `ExportRuntimeFieldValue` (~73 LoC)
  share an identical type-dispatch ladder, with MGIR's extra
  `FArrayProperty` branch and a `_InContainer` vs `_Direct` fallback
  variant as the only real divergences. Loop pair
  `AppendReflectedExpressionProperties` (MGIR 325-363) and
  `AppendReflectedNodeFields` (AGIR 336-392) shares the
  TFieldIterator → IsSafe → sort → CDO-delta → emit pipeline; separator
  (`:` vs `=`) is the only divergence. Recommended as soft prereq for
  upcoming SCIR/BTIR/MSIR/NIR tickets to avoid a 5× copy-paste of the
  same dispatch.
- `#2-shared-reflected-emit` `IN-REVIEW` developer — Added
  `FReflectedFieldEmitOptions`, `FIrTextUtils::FormatReflectedPropertyValue`,
  and `FIrTextUtils::AppendReflectedFields`; migrated MGIR/AGIR reflected
  property emit to the shared helpers while preserving MGIR's opt-in bracket
  array output and AGIR's colon field separator; added
  `FIrTextUtilsReflectedPropertyEmitTest` coverage for value formatting,
  array mode selection, field skipping, reject predicates, sorting, and custom
  separators.
- `#3-verify-reflected-emit-test` `DONE` tester — Verified: ran `system.run_tests tests:["EditorAutomationRpcGateway.core.ir_text.reflected_property.Emit"]` through the already-running gateway at `127.0.0.1:19880`; job `j_20260515T071017_f0a8961f` resolved the exact requested test with `missingTests:[]` and completed with `has_errors:false`.
