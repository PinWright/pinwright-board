---
id: B-crir-compile-failure-not-atomic
title: "controlrig.compile_crir clears RigVM graphs before fallible emission and returns errors without restoring the rig"
status: OPEN
severity: Critical
category: bug
tags: [crir, control-rig, compile, replace, rollback, atomicity, data-loss]
---

# A failed CRIR compile leaves cleared graphs and partial hierarchy/function edits

Replace resolves a model and controller, calls `ClearGraphForReplace`, then begins emission (`Plugins/PinWright/Source/PinWright/Private/CRIR/CRIRCompiler.cpp:214-249`). The clear snapshots all nodes and removes each with `bSetupUndoRedo=false` (`:79-96`). A valid CRIR document can then name a nonexistent unit struct; `AddUnitNodeFromStructPath` fails and returns `CRIR_UNIT_CREATE_FAILED` (`:349-384`) after the graph has already been erased. Later instruction failures likewise leave earlier additions.

The outer `FScopedTransaction` and `ModifyControlRigBlueprint` (`CRIRCompiler.cpp:1470-1472`) do not undo on scope exit. All graph/controller edits explicitly disable per-call undo, and every pass returns immediately on failure (`:1517-1557`) without `Cancel`, editor undo, or explicit restore. Pass 1 can also add hierarchy elements or functions before a Pass-2 graph error. `controlrig.compile_crir` only forwards that error (`Handlers/ControlRig/CRIRCompileHandler.cpp:69-73`). The caller sees failure while the live rig is cleared/partial, and a later save can persist it.

Preflight all model, struct, function, hierarchy-parent, and wire references before the first edit. Then snapshot the RigVM models, hierarchy, and local function library and use the established transaction rollback shape (`PrepareTransactionalSnapshot`, finalized transaction, `ApplyAndCancelTransaction`) on every failed pass. Verify post-rollback topology and hierarchy against the snapshot. Extend must also remove its partial additions.

**Workaround:** compile only into a disposable duplicate; after an error, reload the original package from disk before any save.

## Related

- Catalog: `partial-mutation-without-complete-rollback`, `partial-nonatomic-success`
- `B-crir-replace-duplicates-hierarchy` — Replace idempotency, not failure rollback
- `B-compile-bpir-transaction-ensure` — sibling transaction precedent

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed post-clear `CRIR_UNIT_CREATE_FAILED` and cross-pass early returns with no restore; no editor, build, or test was run.
