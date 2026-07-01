---
id: E-ircore-shared-local-id-helper
title: "Extract `%nN` local-id formatter to IrCore (4 IR families now duplicate it)"
status: DONE
severity: Low
category: ergonomic
tags: [ir, ircore, refactor, bpir, mgir, msir, crir]
---

# Extract `%nN` local-id formatter to IrCore

Four IR families now use the same `FString::Printf(TEXT("n%d"), Counter)`
pattern to allocate per-block local identifiers (`%n0`, `%n1`, ...) during
decompile:

- `Source/EditorAutomationRpcGateway/Private/Decompiler/BpirDecompiler.cpp:165`
- `Source/EditorAutomationRpcGateway/Private/MGIR/MGIRDecompiler.cpp:60`
- `Source/EditorAutomationRpcGateway/Private/MSIR/MSIRDecompiler.cpp:195`
- `Source/EditorAutomationRpcGateway/Private/CRIR/CRIRTextEmitter.cpp` —
  `FCRIRTextEmitter::MakeLocalId`

AGIR uses a richer mnemonic-based form (e.g. `%pose0`, `%link0`) and is not a
target for this extraction; the `%nN` style applies to BPIR/MGIR/MSIR/CRIR.

The CRIR Phase A sprint's reuse reviewer flagged this as the threshold case
("3+ IR families using the pattern justifies extraction") and recommended
deferring to a follow-up because the touch-set spans existing files outside
the CRIR sprint scope.

**Fix:** Add a tiny helper to `Source/EditorAutomationRpcGateway/Private/IrCore/IrTextUtils.h`:

```
namespace FIrTextUtils
{
    inline FString FormatNumericLocalId(int32 Counter)
    {
        return FString::Printf(TEXT("%%n%d"), Counter);
    }
}
```

Then update the four call sites to consume the helper. Each call site is a
one-line replacement; verify the test round-trip on each IR family after
swap (existing tests cover all four). No grammar change — the on-the-wire
text is identical.

## History
- `#1-initial-spec` `OPEN` developer — Filed during CRIR Phase A
  (`F-control-rig-ir-language`) sprint review. The reuse reviewer flagged
  CRIR as the 4th duplicator of this pattern and recommended IrCore
  extraction once 3+ IR families share the shape; the threshold is now met.
  Deferred from the CRIR sprint to keep that sprint's diff scoped to CRIR
  itself.
- `#2-extracted-helper` `IN-REVIEW` developer — Added
  `FIrTextUtils::FormatNumericLocalId(int32)` (declaration in
  `Public/IrCore/IrTextUtils.h`, definition in
  `Private/IrCore/IrTextUtils.cpp`). Updated the four call sites:
  `BpirDecompiler.cpp` (FEntryState::AllocValueName), `MGIRDecompiler.cpp`
  (NextFallbackName fallback), `MSIRDecompiler.cpp` (Node entry naming),
  and `CRIRTextEmitter::MakeLocalId` (now delegates). The proposed code in
  the ticket body used `TEXT("%%n%d")` which would have produced `%n0`;
  corrected to `TEXT("n%d")` because all four call sites emit the bare
  `nN` token — the leading `%` sigil is added by emission-site format
  strings (e.g. CRIR's `EmitUnit` does `%%%s = unit ...`). On-the-wire
  text is identical to pre-refactor output. Added
  `FIrTextUtilsNumericLocalIdTest` in `Tests/Core/TestIrTextUtils.cpp`
  asserting `FormatNumericLocalId(0) == "n0"`, `(7) == "n7"`, `(42) ==
  "n42"`; if the helper format is reverted to `%%n%d` the test fails on
  every assertion because the output starts with `%`.
- `#3-verify-helper-refactor` `DONE` tester — Verified: static check of
  `FormatNumericLocalId` found the helper declared in `Public/IrCore/IrTextUtils.h`,
  defined as `FString::Printf(TEXT("n%d"), Counter)` in
  `Private/IrCore/IrTextUtils.cpp`, consumed by the four claimed call sites
  (`BpirDecompiler.cpp`, `MGIRDecompiler.cpp`, `MSIRDecompiler.cpp`,
  `CRIRTextEmitter.cpp`), and covered by `FIrTextUtilsNumericLocalIdTest`
  assertions for `n0`, `n7`, and `n42`.
