---
id: E-bpir-wildcard-type-keyword-unresolved
title: "BPIR `wildcard` type keyword is documented but not in the type grammar: it resolves as an unknown identifier and logs a warning"
status: OPEN
severity: Low
category: ergonomic
tags: [bpir, types, wildcard, macro, type-grammar]
encounters: 1
lastSeen: 2026-10-02T12:00:00Z
rice: [1, 1, 1, 1]
priority: 8
---

# `wildcard` is a documented BPIR type with no grammar entry

`docs/wiki-src/bpir.types.md` lists `wildcard  # Wildcard (generic)`, and the decompiler
prints a wildcard macro tunnel pin as `wildcard Name` (`PinTypeToBpirType` falls through to
the raw pin category). But `BpirTypeGrammar.cpp` has no `wildcard` row, so
`BpirTypeSpecParser` parses it as `EBpirTypeKind::Unresolved` and
`FCodePinResolver::ConvertTypeSpecToPinType` walks Enum -> Struct -> Class, fails, logs
`LogCodePinResolver: Warning: ConvertTypeSpecToPinType: unresolved identifier 'wildcard'`
and returns false.

It only works by accident: `SetupMacro` (`Compiler/BpirCompiler.cpp`) pre-sets
`PinType.PinCategory = PC_Wildcard` and ignores the return value, so the tunnel pin stays
wildcard. Any other caller that checks the return (or a host that elevates log warnings)
sees a failure for a documented type.

## Repro (source reading, no editor run)

```
entry macro PickWild(bool C) -> (wildcard Out) {
    %a = select(Index: $C, true: 1, false: 2)
    return (Out: %a)
}
```

Compiles with a wildcard `Out` pin and one `unresolved identifier 'wildcard'` warning.
Used by `PinWright.bpir.compiler.select_literal.MacroSelectsStayWildcard`.

## Expected

`wildcard` is a grammar keyword mapping to `PC_Wildcard` with no warning, so the documented
type and the decompiler's output round-trip cleanly.

## History

- `#1-filed` `OPEN` reporter — Found while writing the G03 macro-select regression test: the documented `wildcard` type has no `BpirTypeGrammar` row and resolves through the unresolved-identifier path with a warning; the macro tunnel stays wildcard only because `SetupMacro` pre-sets the category and ignores the failure.
