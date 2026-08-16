---
id: B-mgir-extend-refusal-not-atomic
title: "A refused MGIR Extend leaves partial additions in the material — instructions before the offending one are already emitted, there is no transaction at any granularity, and a retry accumulates instead of converging"
status: OPEN
severity: Medium
category: bug
tags: [mgir, material, compile, atomicity, rollback, partial-application, idempotency, transaction]
---

# `Extend` refuses mid-document, after N-1 expressions have already been created

`FMGIRCompiler` emits as it walks. `CompileBlockInstructions`
(`MGIR/MGIRCompiler.cpp:766-787`) is one pass with a bare `return` on the first error:

```cpp
    for (const FMGIRInstruction& Instruction : Block.Instructions)
    {
        FMGIRCompileResult EmitResult = EmitInstruction(...);
        if (!EmitResult.ErrorCode.IsEmpty())
        {
            return EmitResult;
        }
    }
```

No validation pass runs over `Block.Instructions` before the loop, and nothing undoes instructions
0..N-1 when instruction N fails. Each `EmitInstruction` mutates the asset immediately —
`FMaterialExpressionFactory::Create` allocates the expression outered to the material and adds it to
the live expression collection (`Material/MaterialExpressionFactory.cpp:227-228`, `:254`).

The `Extend` guard is armed per block (`MGIRCompiler.cpp:852-863`, `Emitter.GuardExistingExpressionHandles()`)
but the refusal is raised from inside the per-instruction emit: `MGIR_EXTEND_CANNOT_UPDATE` comes
from `ValidateNewSymbol` (`MGIR/MGIRExpressionEmitter.cpp:159-193`), a **per-symbol check performed
at emit time**, called at the top of every `Emit*`. So a document of N instructions where only the
last names an existing handle emits N-1 expressions into the asset and then returns the refusal.

## There is no transaction, at any granularity

`grep -r Transaction Source/PinWright/Private/MGIR/` returns **nothing** — not whole-document, not
per-instruction. The handler does not wrap it either (`Handlers/Material/MGIRCompileHandler.cpp:63-68`
calls `FMGIRCompiler::Compile` and forwards the error straight out).

This is a striking gap next to the sibling IR. `FScopedTransaction` appears 173 times in 63 files
across the plugin, and the BPIR compiler has **both** a pre-pass and an explicit rollback:
`Compiler/BpirCompiler.cpp:2515-2525` runs `ValidateNoDuplicatePlainEntryEvents(Blocks, ...)` over
the whole document *before* opening `FScopedTransaction`, and `:3184-3211` defines
`RollbackCreatedState()` which deletes every created node by GUID when errors accumulated. It even
has a test — `PinWright.bpir.compiler.errors.AtomicRollback`
(`Tests/Bpir/TestCompilerErrors.cpp:134-138`), listed as "Full" coverage in
`Docs/bpir-test-matrix.md:138`. MGIR has neither.

## Cross-block leakage too

`MGIRCompiler.cpp:938-961` returns on the first failing block, so in a multi-entry document the
blocks already compiled stay applied **and finalized** — `FinalizeMaterial` runs per block
(`:865-869`) and with the default `save: true` writes the `.uasset`. A refused document can
therefore leave a partially-extended material **on disk**.

## Not a regression — but the sibling proves the bar

Every error inside `EmitInstruction` (`MGIRCompiler.cpp:571-764`) has the same shape: `MGIR_CREATE_FAILED`
(`:617-619`), `MGIR_INVALID_ATTRIBUTE` (`:627`, `:634`), `MGIR_INVALID_CONSTANT` (`:650`),
`MGIR_UNSUPPORTED_OPCODE` (`:752-754`), and the generic funnel at `:757-760`. Both post-loop wiring
passes bail the same way (`:807-810`, `:820-823`) — those fail *after* every expression in the
document has been created. So the `Extend` refusal is consistent with the whole surrounding design,
which is why this is filed at Medium rather than as a regression.

## Why the existing test does not catch it

`Tests/Material/TestMGIRExtendExistingHandle.cpp` is the only `Extend` coverage, and its seed
document (`:70-75`) is a single constant plus an output — **the offending instruction is instruction
#1**, so nothing has been emitted before the refusal fires. That is exactly why `:156` passes:

```cpp
    TestEqual(TEXT("no duplicate expression was created"), CountExpressions(Material), 1);
```

Add one *new* symbol ahead of the offending line and the same assertion becomes 2. The file's own
header (`:10-11`) already says these tests assert the expression graph rather than the return value;
the repro is that claim extended to a two-instruction document.

## Which rules this straddles

- `rpc-design.md:41` §1 "report only what happened" — the refusal reports a pure error while N-1
  additions committed.
- `rpc-design.md:181` §8 — *"Any write-side verb must be idempotent … a client retry must converge to
  the same final state rather than accumulate."* A partially-applied `Extend` makes each retry hit
  `MGIR_DUPLICATE_SYMBOL` / `MGIR_EXTEND_CANNOT_UPDATE` **earlier** than the last, which accumulates
  rather than converging. `rpc-design.md` has no atomicity/rollback section at all.

## Documentation implies the opposite

`wiki-src/material.mgir.md:69` (generated at `Saved/PinWright/wiki/material.mgir.md:71`) says a
decompile -> edit -> compile round trip means *"every symbol names something that already exists, and
`Extend` **refuses the whole document**"*. "Refuses the whole document" reads as an atomicity
guarantee. Nothing on that page, on `material.compile_mgir`, or in the handler summary
(`MGIRCompileHandler.cpp:39`) says what happens to instructions already applied.

## Fix shape

A pre-pass immediately before the emit loop, between `MGIRCompiler.cpp:773` and `:775`: walk
`Block.Instructions`, collect each `ResultName`, and reject the whole document if any collides with
`PreExistingHandles` — or with another instruction's result name, which also catches
`MGIR_DUPLICATE_SYMBOL` up front. `GuardExistingExpressionHandles()` already runs before the loop
(`:855` / `:903`), so the index is populated in time. It needs one new read-only accessor on the
emitter, because `PreExistingHandles` (`MGIRExpressionEmitter.h:116`) and `ValidateNewSymbol` (`:120`)
are both private.

**Scope honestly:** that closes only the `Extend`-refusal class. `MGIR_INVALID_CONSTANT`,
`MGIR_UNSUPPORTED_OPCODE`, `PROPERTY_NOT_FOUND` and both wiring passes still fail mid-apply. True
atomicity needs the BPIR treatment — a created-expression list plus rollback, or an
`FScopedTransaction` around `Compile`.

## History
- `#1-partial-additions-survive-a-refusal` `OPEN` reporter — A refused MGIR `Extend` leaves the instructions before the offending one already applied to the material. `CompileBlockInstructions` (`MGIRCompiler.cpp:766-787`) is a single emit-as-you-go pass with a bare `return` on the first error, with no pre-pass over `Block.Instructions` and no undo; each `EmitInstruction` mutates the asset immediately via `FMaterialExpressionFactory::Create` (`MaterialExpressionFactory.cpp:227-228`, `:254`). The `Extend` guard is armed per block (`MGIRCompiler.cpp:852-863`) but `MGIR_EXTEND_CANNOT_UPDATE` is raised per symbol at emit time from `ValidateNewSymbol` (`MGIRExpressionEmitter.cpp:159-193`), so a document whose LAST instruction names an existing handle emits N-1 expressions and then refuses. **There is no transaction at any granularity** — `Transaction` has zero matches across `Private/MGIR/`, and `MGIRCompileHandler.cpp:63-68` does not wrap the call. Worse across blocks: `MGIRCompiler.cpp:938-961` returns on the first failing block, and `FinalizeMaterial` runs per block (`:865-869`) with `save: true` by default, so a refused multi-entry document can leave a partially-extended material **on disk**. Filed Medium, not as a regression: every other mid-document MGIR error has the identical shape (`:617-619`, `:627`, `:634`, `:650`, `:752-754`, the generic funnel at `:757-760`, and both post-loop wiring passes at `:807-810` / `:820-823`), so this is a design gap rather than a new break.
- `#2-the-sibling-ir-already-does-this-right` `OPEN` reporter — The bar is already set inside this plugin, which is what makes the gap actionable rather than aspirational. `FScopedTransaction` appears 173 times in 63 files; the BPIR compiler runs `ValidateNoDuplicatePlainEntryEvents(Blocks, ...)` over the whole document *before* opening its transaction (`BpirCompiler.cpp:2515-2525`) and defines `RollbackCreatedState()` deleting every created node by GUID when errors accumulated (`:3184-3211`), with a regression test `PinWright.bpir.compiler.errors.AtomicRollback` (`Tests/Bpir/TestCompilerErrors.cpp:134-138`) recorded as Full coverage in `Docs/bpir-test-matrix.md:138`. MGIR has neither the pre-pass nor the test. Also worth recording: the existing `Extend` coverage cannot fail on this symptom — `Tests/Material/TestMGIRExtendExistingHandle.cpp` seeds a document whose offending instruction is #1 (`:70-75`), so nothing has been emitted when the refusal fires, which is precisely why its `CountExpressions(Material) == 1` assertion at `:156` passes; adding one new symbol ahead of the offending line turns that into 2.
- `#3-rules-straddled-and-the-doc-that-implies-otherwise` `OPEN` reporter — Two `rpc-design.md` rules are in tension with current behaviour. §1 "report only what happened" (`:41`) — the refusal reports a pure error while N-1 additions committed. §8 (`:181`) requires write-side verbs to be idempotent so a client retry "converges to the same final state rather than accumulates"; a partially-applied `Extend` makes each retry fail EARLIER than the last, which is the opposite. `rpc-design.md` has no atomicity/rollback section to hang this on. Meanwhile the published docs imply the guarantee that does not exist: `wiki-src/material.mgir.md:69` (generated at `Saved/PinWright/wiki/material.mgir.md:71`) says `Extend` "refuses the whole document", and nothing on that page, on `material.compile_mgir`, or in the handler summary (`MGIRCompileHandler.cpp:39`) says what happens to instructions already applied. Fix shape: a pre-pass between `MGIRCompiler.cpp:773` and `:775` collecting every `ResultName` and rejecting the document on a collision with `PreExistingHandles` or with another instruction's result name — `GuardExistingExpressionHandles()` already runs before the loop (`:855`/`:903`) so the index is populated, but it needs a new read-only accessor since `PreExistingHandles` (`MGIRExpressionEmitter.h:116`) and `ValidateNewSymbol` (`:120`) are private. That closes only the `Extend`-refusal class; the constant, opcode, property and wiring failures still apply mid-document, and true atomicity needs the BPIR treatment.
