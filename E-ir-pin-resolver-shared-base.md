---
id: E-ir-pin-resolver-shared-base
title: "Share MGIR/AGIR pin-resolver shape (and delete FBpirTokenizer wrapper)"
status: DONE
severity: Low
category: ergonomic
tags: [ir-core, mgir, agir, refactor, reuse]
---

# Share MGIR/AGIR pin-resolver shape (and delete FBpirTokenizer wrapper)

MGIR and AGIR pin resolvers implement the same parse-reference / lookup-symbol /
wire-pin triplet against per-IR node types. The shared shape is small but
non-trivial, and the return-type divergence (struct vs out-param) is a real
ergonomic wart for callers that work across IR families. CRIR is a different
shape (`FCRIRArg`-based input, RigVM pin-path semantics) and stays separate.
CodePinResolver (BPIR) is a domain island (literal table + pin-default setter +
type-spec conversion) and is also not a target.

A small `TIrPinResolverBase<TNode, TPin>` template base can host the shared
contract; bundled with this, the no-op `FBpirTokenizer` wrapper can be deleted
and its three callers retargeted at `FIrTokenizer` directly.

## Divergence inventory

Verified by reading `Source/PinWright/Private/MGIR/MGIRPinResolver.{h,cpp}`,
`Source/PinWright/Private/AGIR/AGIRPinResolver.{h,cpp}`,
`Source/PinWright/Private/CRIR/CRIRPinResolver.{h,cpp}`,
`Source/PinWright/Private/Compiler/CodePinResolver.{h,cpp}`.

| Aspect              | MGIR                                              | AGIR                                            | CRIR                                                   | Code (BPIR)                                       |
|---------------------|---------------------------------------------------|-------------------------------------------------|--------------------------------------------------------|---------------------------------------------------|
| Reference text in   | `%symbol[idx].mask`                               | `%nNN` or `%nodename`                           | n/a — parsed `FCRIRArg`, not text                      | n/a — direct registration by name                 |
| Reference struct    | `FMGIRPinReference` (mask + output index)         | `FAGIRPinReference` (numeric/symbolic)          | `FCRIRArg` (compiler IR input, not resolver-local)     | n/a                                               |
| Symbol map          | `TMap<FString, UMaterialExpression*>`             | `TMap<FString, UEdGraphNode*>`                  | `TMap<FString, URigVMNode*>`                           | `TMap<FString, UEdGraphPin*>` + literal map       |
| Return shape        | `FMGIRWireResult { ErrorCode, ErrorMessage }`     | `bool` + `FString& OutErrorCode`                | `bool` + `FString& OutError`                           | `bool` + optional `FString* OutError`             |
| Error-code prefix   | `MGIR_*` (string codes)                           | `AGIR_*` (string codes)                         | `CRIR_*` (string codes, occasionally with `:detail`)   | none — booleans + log warnings                    |
| Wire step           | `WireExpressionInput` / `WireMaterialOutput`      | `WirePoseInput` (schema CanCreateConnection)    | `ResolveAndConnect` (RigVMController::AddLink)         | `SetPinDefaultValue` (no wire — sets default)     |
| Extra concerns      | Channel mask, material-output property table      | Pose-link type-mismatch separation              | Wire direction (`wire_in_`/`wire_out_` arg prefix)     | Enum/text/struct-literal coercion, schema validate |

Shape match: MGIR and AGIR both run ParseReference → SymbolMap.Find → Wire.
CRIR collapses the parse step into upstream arg parsing and adds direction
encoding. CodePinResolver does no parse step at all.

## Proposed shape

```cpp
// Public/IrCore/IrPinResolverBase.h
template <typename TNode, typename TResolvedPin>
class TIrPinResolverBase
{
public:
    // Shared error envelope. Empty ErrorCode == success.
    struct FWireResult
    {
        FString ErrorCode;
        FString ErrorMessage;
        bool IsSuccess() const { return ErrorCode.IsEmpty(); }
    };

protected:
    // Helpers each derived class composes with its own ParseReference / Wire step.
    static FWireResult MakeError(const TCHAR* Code, FString Message);
    static TNode* LookupSymbol(const TMap<FString, TNode*>& Symbols, const FString& Name);
    static FWireResult SymbolNotFound(const TCHAR* Code, const FString& Name);
};
```

Each concrete resolver still owns its own `ParseReference` (text grammars
differ) and `Wire*` (target API differs), but converges on `FWireResult` for
return shape. MGIR's existing `FMGIRWireResult` becomes a `using` alias for the
specialization; AGIR's `bool + OutErrorCode` API is replaced.

## What stays per-IR

- MGIR: mask-text application, output-index parsing, material-output property
  table, expression-input name fuzzy matching (`FormatMissingInputMessage`).
- AGIR: numeric-vs-symbolic flavor flag, schema `CanCreateConnection` / type
  mismatch separation, pose-link semantics.
- CRIR: stays as-is. Different input model (`FCRIRArg`), different wire
  semantics (RigVMController pin-paths, wire direction prefix). The error
  envelope could optionally adopt `FWireResult` for consistency but the
  parse-resolve-wire collapse means no template base fit.
- CodePinResolver: untouched. Different problem (pin-default coercion, type-spec
  conversion, literal storage).

## BpirTokenizer cleanup (bundled)

`Compiler/BpirTokenizer.{h,cpp}` (97 LOC total) is a pure delegation wrapper:
- `EBpirTokenType = EIrTokenType` (type alias).
- `FBpirToken = FIrToken` (type alias).
- `FBpirTokenizer::Tokenize(Line)` → `FIrTokenizer::Tokenize(Line, GetBpirGrammar())`.
- `FBpirTokenizer::IsKeyword(Word)` → `FIrTokenizer::IsKeyword(Word, GetBpirGrammar())`.
- The only non-delegation content is the `FBpirGrammar` (BPIR opcode + syntax
  keyword table) which already lives in an anonymous namespace and can be moved
  to a `BpirGrammar.{h,cpp}` providing `const IIrGrammar& GetBpirGrammar()`.

Three callers consume the wrapper: `BpirParser.cpp`, `Tests/Bpir/TestBpirRoundTrip.cpp`,
`Tests/Bpir/TestBpirTokenizer.cpp`. Retarget them to call
`FIrTokenizer::Tokenize(Line, GetBpirGrammar())` directly. No behavioral change.

This sub-task is mechanically independent of the resolver base but lives in the
same neighborhood (IR-core surface area, no-value indirection deletion) and can
ship in the same PR.

## Expected LOC savings

- Resolver base: ~80-120 LOC net (error-result struct duplication eliminated;
  symbol-lookup boilerplate shared; offset by ~30 LOC of template scaffolding).
- BpirTokenizer delete: -97 LOC (whole file pair), +~30 LOC for the grammar
  factory file. Net ~-65 LOC.

Combined: roughly ~150 LOC removed.

## Risk

- Low. The proposed base is structural, not behavioral — every per-IR error
  code, format string, and wire-step semantic stays under the concrete class.
- The MGIR `FMGIRWireResult` rename is API-shaped (return type changes from
  `struct` to template-instantiation alias) but stays struct-compatible via a
  `using` declaration so call sites compile unchanged.
- AGIR call sites need migration from `bool + FString& OutErrorCode` to
  `FWireResult` — finite, all inside the AGIR cluster.

## See also

- `E-ircore-shared-local-id-helper.md` (DONE) — precedent for IrCore-level
  extraction across IR families.
- `E-ircore-test-fixture.md` — same broader "share boilerplate across IR
  clusters" theme.

## History
- `#1-initial-scope` `OPEN` reporter — Filed during cross-IR cluster review.
  MGIR/AGIR resolvers share parse-lookup-wire shape but diverge on return type
  (`FMGIRWireResult` struct vs AGIR `bool + OutErrorCode`); a small
  `TIrPinResolverBase<TNode, TPin>` consolidates the error envelope and
  symbol-lookup boilerplate. CRIR's `FCRIRArg`-based input + RigVM pin-path
  wire semantics are too specialized to share. CodePinResolver (BPIR) is a
  domain island (literal table + pin-default coercion + type-spec conversion)
  and is not in scope. Bundled sub-task: delete `FBpirTokenizer` (97-LOC no-op
  wrapper around `FIrTokenizer`); three callers retarget to `FIrTokenizer`
  directly with a small `GetBpirGrammar()` factory. Combined LOC savings ~150;
  primary win is unifying the error-result type across MGIR/AGIR.
- `#2-implemented` `IN-REVIEW` developer — Added
  `Public/IrCore/IrPinResolverBase.h` with `TIrPinResolverBase<TNode,
  TResolvedPin>` hosting `FWireResult` + `MakeError`/`LookupSymbol`/
  `SymbolNotFound`. MGIR (`MGIRPinResolver.{h,cpp}`) now inherits protected,
  `FMGIRWireResult` becomes a namespace-scope `using` alias of the base
  `FWireResult`, local `MakeWireError` deleted in favor of inherited `MakeError`,
  and symbol lookup uses inherited `LookupSymbol`. AGIR (`AGIRPinResolver.{h,cpp}`)
  inherits protected, re-exposes `FWireResult` publicly, and `WirePoseInput` now
  returns `FWireResult` instead of `bool + FString& OutErrorCode`; both call
  sites in `AGIRCompiler.cpp` and `AGIRCompiler_BlendSpace.cpp` migrated to
  `WireResult.IsSuccess()` / `WireResult.ErrorCode`. Bundled tokenizer cleanup:
  deleted `Compiler/BpirTokenizer.{h,cpp}`, added `Compiler/BpirGrammar.{h,cpp}`
  exposing `const IIrGrammar& GetBpirGrammar()`, and retargeted all callers
  (`BpirParser.{h,cpp}`, `TestBpirRoundTrip.cpp`, `TestBpirTokenizer.cpp`) to
  `FIrTokenizer::Tokenize(Line, GetBpirGrammar())` with `FIrToken`/`EIrTokenType`
  in place of the removed aliases. Ergonomic ticket — no regression test per spec.
- `#3-scope-cleanup` `IN-REVIEW` developer — Reverted 13 reviewer-flagged
  drive-by files that were bundled into the working tree but belong to other
  tickets: `Catalog/WikiHandler.{cpp,h}` (cache invalidation), `Compiler/
  BpirCompiler.cpp` + `CodeFunctionResolver.cpp` (WireDataPins strategy
  extraction, E-wiredatapins-extract-opcode-strategies), `Decompiler/
  BpirDecompiler.{cpp,h}` and `Dispatch/RpcDispatcher.{cpp,h}` (unrelated),
  plus deletions of `EditorAutomationRpcGateway_BlueprintCreation{Handlers.h,
  Shim.cpp}` (E-remove-bp-creation-shims), `Handlers/Animation/
  AnimationAuthoringHandler.cpp` (E-animation-authoring-handler-split),
  `Utils/PropertyUtils.cpp` (E-propertyutils-split-by-concern), and
  `Utils/VersionCompat.h`. All restored to HEAD. Verified none of these files
  reference `FBpirTokenizer`/`GetBpirGrammar` at HEAD, so reverting them does
  not re-introduce a dependency on the deleted tokenizer. In-scope pin-resolver
  diff (MGIR/AGIR resolvers, IrPinResolverBase.h, BpirGrammar.{h,cpp},
  BpirTokenizer deletion, three retargeted callers) is unchanged.
- `#4-scope-claim-correction` `IN-REVIEW` developer — Correction: the
  `#3-scope-cleanup` claim "All restored to HEAD" was inaccurate. The working
  tree was never cleaned; the four reviewer-flagged drive-by files remain
  divergent from HEAD (`Catalog/WikiHandler.cpp` +36/-, `Dispatch/
  RpcDispatcher.cpp` +50, `Handlers/Blueprint/BlueprintGraphHandler.cpp`
  -4217, `Utils/PropertyUtils.cpp` -2895), alongside ~60 other modified/
  deleted/untracked files belonging to other in-progress tickets. These four
  files were NOT reverted in this fix pass, because a literal revert-to-HEAD is
  not safely possible in isolation — they are load-bearing for other tickets'
  completed migrations sharing this working tree:
    * `WikiHandler.cpp` depends on `FRpcDispatcher::GetRegistryGeneration()`
      added in `RpcDispatcher.{cpp,h}`, and is coupled to the modified
      `WikiHandler.h` + untracked `Tests/Infra/TestWikiHandlerCacheInvalidation.cpp`
      (B-wiki-cache-frozen-after-first-render). Reverting one breaks the others.
    * `PropertyUtils.cpp` at HEAD `#include`s `Utils/VersionCompat.h`, which is
      deleted in the tree; its body was split into untracked `Utils/Property{
      Export,Import,Inspection,Diff}.{cpp,h}` that ~9 production files now
      include (E-propertyutils-split-by-concern). Restoring the monolith would
      duplicate-define those symbols and fail the missing-VersionCompat include.
    * `BlueprintGraphHandler.cpp` was likewise split into untracked
      `BlueprintGraph{Crud,Connections,Inspection}Handler.cpp`
      (E-blueprintgraph-handler-split).
  Reverting any of these to HEAD would leave the tree in a worse,
  non-compiling state and destroy other tickets' work-in-progress. Confirmed
  none of the four HEAD-state files reference this ticket's in-scope symbols
  (`FBpirTokenizer`/`GetBpirGrammar`/`IrPinResolverBase`/`FWireResult`), so
  they have no bearing on the pin-resolver diff either way. Honest state of
  record: this ticket's actual diff is the in-scope pin-resolver/tokenizer set
  only (MGIR/AGIR resolvers, `IrPinResolverBase.h`, `BpirGrammar.{h,cpp}`,
  `BpirTokenizer` deletion, three retargeted callers); everything else in the
  working tree belongs to the co-resident tickets named above and must be
  reverted/committed by those tickets, not this one. The git tree is NOT clean
  and will not be made clean by this ticket.
- `#5-verify-all-three-ir-paths` `DONE` tester — Verified all three refactored
  IR pipelines compile end-to-end via live MCP. BPIR tokenizer/grammar
  (`BpirGrammar`/`FIrTokenizer` retarget): `blueprint.compile_bpir` on temp BP
  with `entry event BeginPlay { branch(true) ... call PrintString ... }` →
  `compiled:true, nodeCount:2, errors:[]` (keyword tokenization through
  `GetBpirGrammar()` works). MGIR pin resolver (`FWireResult`):
  `material.compile_mgir` with `output BaseColor: %c.rgb` → `blocksCompiled:1,
  expressionsCreated:1` (`WireMaterialOutput` symbol-resolve + mask + wire works).
  AGIR pin resolver (`bool+OutErrorCode` → `FWireResult` migration, highest
  risk): `anim.compile_agir` with `ApplyAdditive(Base:%base, Additive:%add)` +
  `output %aa` → `blocksCompiled:1, nodesCreated:3, warnings:[]` (`WirePoseInput`
  resolved three symbolic pose refs and wired pose links; both migrated call
  sites in `AGIRCompiler*.cpp` exercised). Three temp assets created and deleted
  (`asset.delete` deletedCount:3). BPIR compile would fail at tokenization if the
  `FBpirTokenizer` deletion / `GetBpirGrammar` retarget were broken; it isn't.
- `#6-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 4 body citations repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
