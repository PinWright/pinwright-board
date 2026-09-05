---
id: B-animation-create-state-machine-failure-not-atomic
title: "animation.create_state_machine can return an error after leaving a new partial state machine in the Anim Blueprint"
status: IN-REVIEW
severity: High
category: bug
tags: [animation, state-machine, rollback, atomicity, partial-mutation, false-failure]
---

# `create_state_machine` mutates before validating the whole nested request

The handler creates the state-machine node and inner graph first (`Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/AnimationHandler.cpp:667-685`), then creates requested states one by one (`:687-727`) and transitions one by one (`:751-808`). A later missing transition endpoint returns `SOURCE_STATE_NOT_FOUND` or `TARGET_STATE_NOT_FOUND` (`:774-788`) after the machine and all earlier states already exist; a later state/transition factory failure has the same shape. There is no `FScopedTransaction`, snapshot, rollback, or cleanup in this handler.

Because `MarkBlueprintAsStructurallyModified` is reached only on success (`:811`), the error looks like a refusal even though live graph objects were already added. A later unrelated structural edit/save can persist the partial machine.

Preflight every nested element and all transition endpoint names before creating the machine. Then wrap construction in the established graph transaction/snapshot rollback shape and remove/restore every created graph/node on any reported construction failure. A regression test must force a failure after at least one graph mutation and assert the Blueprint topology and package dirty state are unchanged after the error.

**Workaround:** validate endpoints yourself and build via the fine-grained authoring verbs; reload the package after any one-shot error.

## Fix

`AnimationHandler.cpp` now preflights every `states[]` and `transitions[]` element, resolves all transition endpoints before the first mutation, and wraps construction in `FScopedTransaction` plus `BlueprintGraphSnapshot` rollback. Transactional property diffs, newly-created graph topology, and the original package dirty state are restored on a reported construction or verification failure. Entry-node wiring remains best-effort as before. The handler harness adds `PinWright.animation.create_state_machine.RollbackOnMidwayFailure`; a private development-test seam forces the second state factory call to fail after the machine and first state exist, then the test compares pre/post graph snapshots and dirty state. Static source review only; no editor, build, or automation run was performed.

## Related

- Catalog: `partial-mutation-without-complete-rollback`, `partial-nonatomic-success`
- `B-create-state-machine-exec-failed` — earlier dead-dispatch implementation, not atomicity
- `B-declared-param-guard-blind-to-nested-keys` — separately owns discarded documented nested keys

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed a missing later transition endpoint returns after graph creation with no rollback; no editor, build, or test was run.
- `#2-atomic-preflight-rollback` `IN-REVIEW` developer — Changed `AnimationHandler.cpp` to preflight nested state/transition input and apply transaction plus graph-snapshot rollback around state-machine construction; added the handler-harness rollback regression test. Static source review only; no editor, build, or test was run.
- `#3-post-mutation-regression` `IN-REVIEW` developer — Corrected the regression to force a second-state factory failure after real graph mutation, assert topology and package dirty-state restoration, and preserved the prior best-effort entry-wiring behavior. Static source review only; no editor, build, or automation run was performed.
