---
id: B-agir-compile-failure-not-atomic
title: "anim.compile_agir clears the target before fallible emission and does not restore it when compilation fails"
status: IN-REVIEW
severity: Critical
category: bug
tags: [agir, animation, compile, replace, rollback, atomicity, data-loss]
---

# A failed AGIR compile leaves the target cleared or partially rebuilt

`FAGIRCompiler::Compile` parses the document and loads the target, but its remaining fallible work is not preflighted. At filing baseline plugin commit `5818176c`, the interface-manifest pre-pass could mutate `ImplementedInterfaces` (`Plugins/PinWright/Source/PinWright/Private/AGIR/AGIRCompiler.cpp:1609-1659`), then Replace called `ClearTargetForReplace` (`:1662-1665`), whose whole body was `ClearAnimGraph` (`:1565-1572`). Only after that destructive clear did it compile blocks (`:1677-1706`). All cited `AGIRCompiler.cpp` ranges in this paragraph are from filing baseline plugin commit `5818176c`.

A syntactically valid `anim_node` naming a missing or non-`UAnimGraphNode_Base` class is a concrete failure path: at filing baseline plugin commit `5818176c`, `CompileCallInstruction` returned `AGIR_CLASS_NOT_FOUND` (`AGIRCompiler.cpp:1041-1067`) after the target was cleared. Any later block error had the same shape, and Extend left whatever earlier nodes/interfaces were added. The RPC handler simply relayed the error (`Handlers/Animation/AGIRCompileHandler.cpp:70-86`); it owned no snapshot, transaction, or rollback. A caller observed an error but the live Anim Blueprint was already empty or partial, and a later unrelated save could persist that state. All cited `AGIRCompiler.cpp` ranges in this paragraph are from filing baseline plugin commit `5818176c`.

The compile must be atomic. Preflight every resolvable class, graph, symbol, and interface before mutation; then snapshot every graph/interface the compile can touch and wrap mutation in the established `FScopedTransaction` + `PrepareTransactionalSnapshot` / `ApplyAndCancelTransaction` failure shape. On every error, restore and compare the pre-compile topology before returning. Cover both Replace deletion and Extend additions in a regression test.

**Workaround:** compile into a disposable duplicate, or reload the clean package from disk immediately after any AGIR error and before saving.

## Related

- Catalog: `partial-mutation-without-complete-rollback`, `partial-nonatomic-success`
- `B-compile-bpir-transaction-ensure` — sibling BPIR rollback contract
- `B-mgir-extend-refusal-not-atomic` — sibling IR partial-emission defect

## Fix

The report was TRUE: after target load, `FAGIRCompiler::Compile` changed implemented interfaces, cleared Replace targets, and emitted blocks before later error returns, with no rollback boundary. The compiler now captures the established all-graph `BlueprintGraphSnapshot`, opens one `FScopedTransaction`, snapshots the Anim Blueprint before the first mutation, and on any interface or block failure applies/cancels the transaction, removes non-transactional graph additions, and restores a previously clean package's dirty flag. A false `ImplementNewInterface` result is reported as `AGIR_INTERFACE_MUTATION_FAILED`. The transaction commits before an optional save on success.

Files changed: `Source/PinWright/Private/AGIR/AGIRCompiler.cpp`, `Source/PinWright/Private/Utils/BlueprintGraphSnapshot.h`, `Source/PinWright/Private/Tests/Assets/TestAGIRCompileAtomicity.cpp`, and `Docs/wiki-src/anim.md`. Tests: `PinWright.anim.agir.CompileFailureAtomicReplace` and `PinWright.anim.agir.CompileFailureAtomicExtend`; each seeds the tracked `ALI_ItemAnimLayers` interface and a real implementation graph/node baseline, applies the distinct native `UAnimLayerInterface` manifest entry, emits one valid node, forces a later `AGIR_CLASS_NOT_FOUND`, and asserts case-sensitive decompiled text plus exact implemented-interface identity, interface graph/node topology, absence of the newly requested interface, and package dirty state are unchanged. `PinWright.anim.agir.CompileInterfaceMutationFailure` creates a real function-name collision, verifies `AGIR_INTERFACE_MUTATION_FAILED`, and asserts the pre-call graph state remains unchanged.

Deliberately not changed: no separate class/symbol preflight pass was added. Duplicating every current and future emission check would create a second compiler; the snapshot-plus-transaction boundary makes all post-load failure paths atomic without allowing the two validation implementations to drift. No Unreal build, editor, or automation run was performed under the worker brief.

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed Replace clear and post-clear `AGIR_CLASS_NOT_FOUND` return with no rollback; no editor, build, or test was run.
- `#2-atomic-compile-rollback` `IN-REVIEW` developer — Wrapped AGIR mutation in the shared transaction plus all-graph snapshot rollback contract, restored clean-package state on failure, documented the atomic contract, and added late-failure Replace and Extend regression coverage. Static diff checks only; compile and automation execution are deferred to the aggregate verification phase.
- `#3-interface-manifest-coverage` `IN-REVIEW` developer — Strengthened both late-failure tests with the tracked `ALI_ItemAnimLayers` anim-layer interface fixture and exact interface/graph topology comparisons; no null-transactor or unattended-host simulation was added because the supported editor commandlet initializes a transactor.
- `#4-interface-rollback-baseline` `IN-REVIEW` developer — Seeded the tracked `ALI_ItemAnimLayers` implementation graph with a node, then had both late-failure tests add the distinct native `UAnimLayerInterface` through AGIR before `AGIR_CLASS_NOT_FOUND`; added the `ImplementNewInterface` false-result rollback error and corrected the snapshot comment to its actual graph collections. No null-transactor or unattended-host simulation was added.
- `#5-interface-mutation-error-coverage` `IN-REVIEW` developer — Added a structural function-name collision regression for the tracked `ALI_ItemAnimLayers` interface so the false `ImplementNewInterface` return is exercised and its `AGIR_INTERFACE_MUTATION_FAILED` rollback contract is checked. Supported editor transaction behavior remains the only exercised host path; no null-transactor or unattended-host simulation was added.
