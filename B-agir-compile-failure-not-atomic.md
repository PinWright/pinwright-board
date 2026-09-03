---
id: B-agir-compile-failure-not-atomic
title: "anim.compile_agir clears the target before fallible emission and does not restore it when compilation fails"
status: OPEN
severity: Critical
category: bug
tags: [agir, animation, compile, replace, rollback, atomicity, data-loss]
---

# A failed AGIR compile leaves the target cleared or partially rebuilt

`FAGIRCompiler::Compile` parses the document and loads the target, but its remaining fallible work is not preflighted. The interface-manifest pre-pass can mutate `ImplementedInterfaces` (`Plugins/PinWright/Source/PinWright/Private/AGIR/AGIRCompiler.cpp:1609-1659`), then Replace calls `ClearTargetForReplace` (`:1662-1665`), whose whole body is `ClearAnimGraph` (`:1565-1572`). Only after that destructive clear does it compile blocks (`:1677-1706`).

A syntactically valid `anim_node` naming a missing or non-`UAnimGraphNode_Base` class is a concrete failure path: `CompileCallInstruction` returns `AGIR_CLASS_NOT_FOUND` (`AGIRCompiler.cpp:1041-1067`) after the target was cleared. Any later block error has the same shape, and Extend leaves whatever earlier nodes/interfaces were added. The RPC handler simply relays the error (`Handlers/Animation/AGIRCompileHandler.cpp:70-86`); it owns no snapshot, transaction, or rollback. A caller observes an error but the live Anim Blueprint is already empty or partial, and a later unrelated save can persist that state.

The compile must be atomic. Preflight every resolvable class, graph, symbol, and interface before mutation; then snapshot every graph/interface the compile can touch and wrap mutation in the established `FScopedTransaction` + `PrepareTransactionalSnapshot` / `ApplyAndCancelTransaction` failure shape. On every error, restore and compare the pre-compile topology before returning. Cover both Replace deletion and Extend additions in a regression test.

**Workaround:** compile into a disposable duplicate, or reload the clean package from disk immediately after any AGIR error and before saving.

## Related

- Catalog: `partial-mutation-without-complete-rollback`, `partial-nonatomic-success`
- `B-compile-bpir-transaction-ensure` — sibling BPIR rollback contract
- `B-mgir-extend-refusal-not-atomic` — sibling IR partial-emission defect

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed Replace clear and post-clear `AGIR_CLASS_NOT_FOUND` return with no rollback; no editor, build, or test was run.
