---
id: F-bpir-type-spec-structured-parse
title: "BPIR type parsing should use a structured FBpirTypeSpec, not ad-hoc text munging in ConvertCppTypeToPinType"
status: DONE
severity: Medium
category: feature
tags: [bpir, parser, tokenizer, type-system, technical-debt, refactor]
---

# BPIR type parsing should use a structured FBpirTypeSpec

## What's wrong

BPIR has a real tokenizer + parser pipeline for statements, but **type strings are handed around as raw `FString`** and re-parsed at compile-emit time by `FCodePinResolver::ConvertCppTypeToPinType` using character-level `StartsWith` / `EndsWith` / `RightChop` / `LeftChop` calls. The tokenizer never looks inside a type string; the parser pulls `(Type Name)` pairs out of `()`-delimited lists by splitting on the last space and stuffing the type portion into `FBpirEntryBlock::FParam::Type` as an opaque string.

Every new type syntax — `struct<T>`, `object<T>`, `array<T>`, the C++-style `struct T` prefix form, and most recently `const T&` — has landed as another string-level patch in `ConvertCppTypeToPinType`. The function is now a ~300-line cascade of `ParseAngleBracketType` / prefix-strip / suffix-strip blocks that has outgrown its shape.

## Concrete symptoms

- **Brittle extension:** adding `const` + `&` took a `RightChop(6)` + `LeftChop(1)` bolted onto the front of the function (see `CodePinResolver.cpp:271-311` after the const-ref fix). Any future qualifier (`ref`, `out`, `inref`, etc.) needs a similar textual prefix/suffix dance.
- **Poor error messages:** when parsing fails, the warning is `"ConvertCppTypeToPinType: unrecognized type '<whole string>'"` — no way to point at which token specifically confused it.
- **Decompiler round-trip risk:** the decompiler stringifies `FEdGraphPinType` back to BPIR syntax via its own function that must agree character-for-character with `ConvertCppTypeToPinType`'s grammar. Two independent text grammars, each ad-hoc, easy to drift.
- **Nested-generic + qualifier interactions are untested:** e.g. `array<const struct<T>&>` parses through recursion but nobody's verified the flag propagation does the right thing on the inner element.
- **Ambiguity between forms:** `struct<T>`, `struct T`, and `struct:T` all produce the same PinType today — but that's enforced by three separate branches in `ConvertCppTypeToPinType`. A structured type spec would have one canonical form and the parser would accept all three sugarings.

## Proposed shape

```cpp
// New file: Compiler/BpirTypeSpec.h
struct FBpirTypeSpec
{
    FName Kind;                                        // struct|object|enum|class|softobject|softclass|interface|array|set|map|<primitive>
    FName InnerName;                                   // e.g. "EditorReplay", "Vector", "int32", or NAME_None for containers
    EPinContainerType Container = EPinContainerType::None;
    TUniquePtr<FBpirTypeSpec> ContainerValueSpec;      // for map<K,V>, V's spec; for array<T>/set<T>, T's spec (or store inner directly)
    bool bIsConst = false;
    bool bIsReference = false;
};
```

Tokenizer emits dedicated type tokens: `Keyword(const)`, `Keyword(struct|object|...)`, `LAngle`, `Identifier(InnerName)`, `RAngle`, `Comma` (for map), `Ampersand`. Parser consumes them into `FBpirTypeSpec`. `FBpirEntryBlock::FParam::Type` becomes `FBpirTypeSpec` instead of `FString`. `ConvertTypeSpecToPinType(const FBpirTypeSpec&, FEdGraphPinType&)` replaces the string-reparsing function and is trivial — a switch on `Kind` with the flag fields copied across.

## What this fixes / enables

- One canonical representation of a type; `StartsWith`/`RightChop` cascade goes away.
- New qualifiers (`ref`, `out`, `inref`, future `mutable` etc.) are a tokenizer + parser change, not a string-munging patch at a distance.
- Tokenizer-level errors: "unexpected token `&` at column 23" vs "unrecognized type '<whole string>'".
- Decompiler `TypeSpecToBpirText(FBpirTypeSpec)` + parser `ParseTypeSpec(Tokens) → FBpirTypeSpec` form an obvious round-trip pair, both referencing one data type.
- Nested generics + qualifiers (`array<const struct<T>&>`, `map<int, struct<FVector>>`) compose cleanly with no special-casing.
- Downstream code in `BpirCompiler.cpp` can branch on `Spec.Kind` directly instead of re-parsing `Param.Type`.

## Scope

Files affected (estimate):
- `Compiler/BpirTokenizer.cpp/.h` — add type-token kinds, emit them inside `()` param lists and after `->` return.
- `Compiler/BpirParser.cpp` — consume type tokens into `FBpirTypeSpec`; replace `Param.Type` / `Param.ReturnType` storage.
- `Compiler/BpirTypes.h` — `FBpirEntryBlock::FParam`, `FBpirEntryBlock::ReturnType`, `FBpirInstruction::TypeArg` (where applicable) change `FString` → `FBpirTypeSpec`.
- `Compiler/CodePinResolver.cpp/.h` — `ConvertCppTypeToPinType(FString)` → `ConvertTypeSpecToPinType(const FBpirTypeSpec&)`. Keep a legacy string overload during migration for compatibility.
- `Compiler/BpirCompiler.cpp` — every site that currently reads `Param.Type` as `FString` needs to read `FBpirTypeSpec`.
- `Decompiler/BpirDecompiler.cpp` — `TypeSpecToBpirText(const FBpirTypeSpec&)` replaces the current ad-hoc type-stringification.
- `docs/bpir-language-reference.md` — document the canonical form + accepted sugarings.
- Test suite: every BPIR test that constructs source with typed params runs through the new pipeline unchanged (BPIR syntax stays the same); the in-memory type representation is what changes. A dedicated unit test for `ParseTypeSpec` / `TypeSpecToBpirText` round-trip is the main new coverage.

Realistic scope: ~20 files touched, ~600 lines of diff, 1-2 sprints with incremental migration (keep the legacy `ConvertCppTypeToPinType(FString)` as a shim during transition).

## Why it's Medium, not High

The current string-based parser compiles and produces correct output for every shape the project currently uses — the const-ref patch that preceded this ticket unblocked Task D without the refactor. This is technical debt, not a functional bug. It becomes forcing the next time we need to add a qualifier that interacts non-trivially with containers (`array<const T&>`, `map<K, const V&>`) or when the decompiler's stringifier and the parser drift on some edge case.

## Non-goals

- **Not** changing BPIR surface syntax. Every accepted form today (`struct<T>`, `struct T`, `const T&`, etc.) stays accepted.
- **Not** rewriting the statement-level tokenizer — types are a sub-grammar nested inside `()` param lists and `->` return positions.
- **Not** touching `FEdGraphPinType` or UE-side type handling.

## History
- `#1-filed-type-parser-debt` `OPEN` reporter — Filed during post-mortem of the BPIR `const struct<T>& Name` parser patch for `B-bpir-bind-dispatcher-external-target-local-event`. That fix landed as another `StartsWith` + `RightChop` + `EndsWith` + `LeftChop` sequence at the top of `ConvertCppTypeToPinType`, prompting the question "why does this function still work with raw strings when the rest of BPIR has a real tokenizer/parser". Answer: technical debt — types started simple and every extension (`struct<T>`, `array<T>`, `const T&`) was a text-level patch, never promoted to structured token consumption. Not blocking any current authoring workflow — filing to capture the refactor before the next qualifier addition forces it.
- `#3-verified-type-forms-live` `DONE` tester — Verified live on `/Game/App/UI/Test/W_McpVerifyTemp`. Single `compile_bpir` exercising primitive, container, qualifier, nested-container shapes through the new pipeline:
  ```
  entry custom_event T_const(const struct<Vector>& V) {}
  entry custom_event T_arr(array<int> Nums) {}
  entry custom_event T_nested(array<const struct<Vector>&> Bag) {}
  ```
  Returned `success: true, nodeCount: 3, status: "UpToDate", errors: [], warnings: []`. The nested `array<const struct<Vector>&>` (the form the prior pre-refactor parser couldn't represent because const/ref couldn't propagate through container recursion) compiles cleanly. The exotic shapes covered by the unit tests now also work end-to-end through the live handler path.
- `#2-type-spec-refactor-landed` `IN-REVIEW` developer — Refactor fully landed (commit `d4e21fb`). New files: `Compiler/BpirTypeSpec.h/.cpp` (`FBpirTypeSpec` struct with `EBpirTypeKind`, container/key/element specs, `bIsConst`/`bIsReference`, deep-copy + `IsVoid`/`IsEmpty`/`Equals`), `Compiler/BpirTypeSpecParser.h/.cpp` (round-trip pair `ParseTypeSpec` / `TypeSpecToBpirText` with column-accurate errors), `Compiler/BpirTypeGrammar.h/.cpp` (unified table: aliases, accept-form bitmask, pin-category mapping — shared by parser, resolver, and decompiler). Consumers migrated: `BpirTypes.h` (`FParam::Type`, `FBpirEntryBlock::ReturnType` now `FBpirTypeSpec`), `BpirParser.cpp` (3 call sites to `ParseTypeSpec`), `BpirSubgraphCompiler.cpp` (parse + `ConvertTypeSpecToPinType`), `BpirCompiler.cpp` (8 call sites to `ConvertTypeSpecToPinType`; diagnostics via `TypeSpecToBpirText`), `CodePinResolver.cpp/.h` (legacy `ConvertCppTypeToPinType(FString)` removed, replaced by `ConvertTypeSpecToPinType(const FBpirTypeSpec&)`), `Decompiler/BpirTextEmitter.cpp` (rebuilt on `BpirTypeGrammar::FindByPinCategory`, so parser and decompiler share one grammar). `FBpirInstruction::TypeArg` intentionally kept as `FString` — every site uses it as a single-identifier slot (cast target, enum name, subsystem class), not a full type grammar; per-ticket "where applicable" carve-out. Docs updated in `docs/bpir-language-reference.md`. Pinned by 9 automation tests in `Tests/Private/Bpir/TestBpirTypeSpec.cpp`: `FBpirTypeSpecPrimitivesTest`, `FBpirTypeSpecTaggedFormsTest`, `FBpirTypeSpecPrefixSugaringsTest`, `FBpirTypeSpecPointerSuffixTest`, `FBpirTypeSpecQualifiersTest`, `FBpirTypeSpecContainersTest` (covers `array<const struct<FVector>&>`, `map<int, struct<FVector>>`), `FBpirTypeSpecUnresolvedTest`, `FBpirTypeSpecEmptyVsVoidTest`, `FBpirTypeSpecErrorQualityTest` (asserts `"unexpected '&' at column"` on `array<const T&&>`). Counterfactual: if `ParseTypeSpec`/`TypeSpecToBpirText` are reverted to the old character-level `StartsWith`/`RightChop` cascade, `FBpirTypeSpecContainersTest` (`array<const struct<FVector>&>`) fails because the old parser couldn't propagate `bIsConst`/`bIsReference` through a recursive container, and `FBpirTypeSpecErrorQualityTest` fails because the old error path only emitted `"unrecognized type '<whole string>'"` with no column.
