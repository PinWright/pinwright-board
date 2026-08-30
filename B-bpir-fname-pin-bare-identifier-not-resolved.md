---
id: B-bpir-fname-pin-bare-identifier-not-resolved
title: "BPIR decompile emits bare FName identifiers but compile rejects them — round-trip break for PC_Name pin defaults"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompile, compile, pin-default, fname, round-trip]
---

# BPIR decompile emits bare FName identifiers but compile rejects them

For a BP node whose `PC_Name` input pin carries a non-empty `DefaultValue`
(e.g. `SetScalarParameterValue.ParameterName = "AnimateArcFill"`), the
decompiler emits the value as a **bare identifier** without quotes:

```
call SetScalarParameterValue(Target: %n7, ParameterName: AnimateArcFill, Value: %n8)
```

Submitting that exact BPIR back to `blueprint.compile_bpir` fails:

```
[COMPILE_FAILED] Line N: Could not resolve value 'AnimateArcFill' for pin 'ParameterName'
```

Compile only accepts the quoted form (`ParameterName: "AnimateArcFill"`).
Net effect: any decompiled BPIR with FName literals cannot be recompiled
without manual quote-wrapping — a round-trip break.

**Root cause:**
- Decompile side: `FBpirDecompiler::FormatPinDefaultLiteral` in
  `Source/PinWright/Private/Decompiler/BpirDecompiler.cpp:2303-2428`
  handles the non-empty `DefaultValue` branch. PC_String / PC_Text /
  PC_FieldPath are wrapped in `"…"`, but PC_Name falls through to the
  catch-all `return DefaultValue;` at line 2234 ("Numeric, name, etc. —
  return as-is"). The unwired default-equivalent case at line 1857 (`return "None"`)
  is unrelated — that path only fires for unwired pins via `FormatPinDefaultLiteral`.
- Compile side: `FBpirValueResolver::IsLiteral` in
  `Source/PinWright/Private/Compiler/BpirValueResolver.cpp:676`
  recognises quoted strings, booleans, `nullptr`, `EnumName::Value`,
  struct literals, asset paths, and numerics — but **not** bare identifiers
  as FName literals. A bare `AnimateArcFill` falls through `IsLiteral`,
  then `ResolveVariable`, and finally hits the "Unknown value reference
  format" / "Could not resolve value '…' for pin '…'" diagnostic in
  `BpirCompiler.cpp:6420-6433`.

**Session evidence:**
- Asset `/App/App/UI/W_GasLeaksTargetFound`, dump at
  `.editor-automation/asset-dumps/App/App/UI/W_GasLeaksTargetFound/bpir.txt`
  lines 24 and 42 show
  `call SetScalarParameterValue(Target: %n7, ParameterName: AnimateArcFill, Value: %n8)`.
- `compile_bpir` with the dumped text fails on that line; wrapping
  `ParameterName: "AnimateArcFill"` compiles.

**Workaround:** Quote the FName value at the call site — `ParameterName: "AnimateArcFill"`.

**Fix:** Make decompile and compile agree on a single FName surface form.
Two options:
1. Decompile side — wrap non-empty PC_Name `DefaultValue` in quotes inside
   `FormatPinDefaultLiteral` (parallel to the PC_String branch at line 2127).
   `FBpirCompiler::SetPinDefaultValue` already strips quotes when storing
   into FName pins, so a quoted decompile output round-trips cleanly.
2. Compile side — extend `IsLiteral` (or the resolver fallback) to accept
   bare alphanumeric identifiers when the target pin is `PC_Name`, routing
   them to `SetPinDefaultValue` as the FName literal. This matches how the
   byte-pin resolver currently handles enum context, but the symmetry is
   weaker: a bare identifier on a non-Name pin would still need to fail.

Option (1) is the smaller, more localised change and keeps the BPIR grammar
unambiguous (FName literals are quoted, same as strings). Option (2) makes
the textual surface more forgiving but risks shadowing bare identifiers
used elsewhere (variable lookup, etc.).

## History
- `#1-initial-repro` `OPEN` reporter — Decompile of `/App/App/UI/W_GasLeaksTargetFound` emits `ParameterName: AnimateArcFill` for `SetScalarParameterValue`; recompiling the same BPIR fails with `Could not resolve value 'AnimateArcFill' for pin 'ParameterName'`. Compile accepts `ParameterName: "AnimateArcFill"`. Root cause traced to `BpirDecompiler.cpp:2234` (PC_Name falls through "as-is") vs `BpirValueResolver.cpp:673` (`IsLiteral` doesn't recognise bare identifiers as FName literals). Workaround: quote the FName at call sites.
- `#2-quote-fname-default` `IN-REVIEW` developer — `ResolveInputValue`'s non-empty-DefaultValue emit path in `BpirDecompiler.cpp` now wraps PC_Name values in quotes via `BpirStructLiteralUtils::EscapeBpirString` (parallel to the PC_String branch just above) before the catch-all `return DefaultValue;`. `FCodePinResolver::SetPinDefaultValue` continues to unquote on store through its generic quoted-string strip at lines 199-207, so the FName round-trip is symmetric. Round-trip restored; test added in `TestBpirRoundTrip.cpp` (`FBpirRoundTripFNamePinDefaultQuotedTest`) — stamps `AnimateArcFill` on a `UKismetSystemLibrary::MakeLiteralName` PC_Name pin, decompiles, asserts the BPIR contains `Value: "AnimateArcFill"` (quoted) and never the bare identifier, then recompiles onto a fresh BP and confirms the recompiled pin's `DefaultValue` matches the original unquoted form.
- `#3-verified-quoted-fname-roundtrips` `DONE` tester — Verified: `blueprint.decompile` of `/App/App/UI/W_GasLeaksTargetFound` now emits `call SetScalarParameterValue(Target: %n8, ParameterName: "AnimateArcFill", Value: %n9)` (quoted). Pre-fix the same asset's BPIR dump (`.editor-automation/asset-dumps/App/App/UI/W_GasLeaksTargetFound/bpir.txt`) carried bare `ParameterName: AnimateArcFill`, which `compile_bpir` rejected with `Could not resolve value 'AnimateArcFill' for pin 'ParameterName'`. Post-fix the quoted form survives a full surgical `insert_bpir_before_node` round-trip without error.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 2 body citations repointed in place and verified against plugin HEAD `ef8a1f1b`. Both lines repaired. The non-empty-`DefaultValue` branch the body describes is **not** in `FormatPinDefaultLiteral` — it is in `FBpirDecompiler::ResolveInputValue` (Case 2, `:2254`), body `:2303-2428`, quoting at `:2308-2315`, catch-all `return DefaultValue;` at `:2427`. `FBpirValueResolver::IsLiteral` opens at `:676` and its enumeration still matches the body. **Two sub-claims are now false, both this DONE ticket's own fix:** PC_Name no longer falls through (explicitly quoted at `:2421-2424`), and the aside's “line 1857 (`return "None"`)” is now `FormatPinDefaultLiteral`'s PC_Name arm at `:2031-2034` (function `:1987-2069`). Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
