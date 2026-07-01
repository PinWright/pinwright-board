---
id: E-make-struct-error-hint
title: "`make<>` error for non-BlueprintType structs should suggest alternatives"
status: DONE
severity: ""
category: ergonomic
tags: []
---

# `make<>` error for non-BlueprintType structs should suggest alternatives

`make<Vector2D>` produces `"The structure Make Vector 2D is not a BlueprintType"`. The error is accurate but doesn't suggest the workaround: use `call MakeVector2D(X:, Y:)` function instead.

**Proposal:** When a `make<StructName>` fails the BlueprintType check, search for a `Make{StructName}` function and suggest it in the error message.

## History
- `#1-initial-repro` `OPEN` reporter — `make<Vector2D>` failed during minimap marker insertion. Had to undo 9 nodes and retry with function calls.
- `#2-added-native-make-hint` `IN-REVIEW` developer — Added early `CanBeMade(Struct, false)` check in BpirCompiler.cpp MakeStruct case. If struct has `HasNativeMake` metadata, resolves native function via `FindObject<UFunction>` and suggests `call FunctionName(...)`. Otherwise emits generic "not a BlueprintType" error. Returns false to prevent broken node creation.
- `#3-verified-make-vector2d` `DONE` tester — Verified: `make<Vector2D>(X: 1.0, Y: 2.0)` returns error "Struct 'Vector2D' cannot be used with make<>; it has a native make function. Use: call MakeVector2D(...)". Suggests correct alternative.
