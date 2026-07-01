---
id: B-bpir-target-shadowed-by-self-class
title: "BPIR resolves call to self-class match even when Target: forces a different class"
status: DONE
severity: High
category: bug
tags: [bpir, function-resolution, cascade, ergonomic]
---

# BPIR resolves call to self-class match even when Target: forces a different class

The 7-step function-resolution cascade documented in `bpir-compiler-internals.md` puts **Self-Class Lookup at Step 1** and **Target Class Lookup at Step 4**. When the unqualified function name happens to match a UFUNCTION on the self-class hierarchy AND a different UFUNCTION on the target's class, Step 1 wins and emits a node bound to the wrong UFunction — even though the BPIR source explicitly passed `Target: %ref` with `%ref` typed to the target class.

The downstream BP compile then errors out on the signature mismatch (or worse, silently misbinds), and the user has no way to disambiguate in BPIR source. Vanilla UE Blueprint editor avoids this because node creation is scoped by the source-pin's type from the start — the BPIR cascade does not honor `Target:` with comparable priority.

## Repro

Add a UFUNCTION to a subsystem whose name collides with any `UUserWidget` member, e.g. `SetPlaybackSpeed(float)` on a subsystem (collides with `UUserWidget::SetPlaybackSpeed(UWidgetAnimation*, float)`). Compile BPIR from inside a Widget Blueprint:

```
%n0: object<RaceAnalyzerSubsystem> = subsystem<RaceAnalyzerSubsystem>()
call SetPlaybackSpeed(Target: %n0, Speed: 0.5)
```

Expected: resolves to `URaceAnalyzerSubsystem::SetPlaybackSpeed(float)` because `Target:` typed `%n0` as the subsystem.

Actual: Step 1 finds `UUserWidget::SetPlaybackSpeed(UWidgetAnimation*, float)` on the widget's own class hierarchy. BPIR emits a node bound to that overload, and either the BP compile fails ("Speed: pin not found") or BPIR errors with "Could not find target pin 'Speed' on node 'SetPlaybackSpeed'".

## Impact

- Real session was blocked: every place wanting `subsystem->SetPlaybackSpeed(float)` from a Widget BP had to be worked around.
- Workaround required all of: (a) add a wrapper function on the widget, (b) populate it via BPIR with a placeholder call to a different subsystem method, (c) `blueprint.graph.replace_node` to swap the node's UFunction to the actual target method (the `target` param of `replace_node` does accept the right resolution), (d) `blueprint.graph.connect_pins` to rewire the argument. Four RPCs for what should be a single `call` line.
- Forces C++ to use awkward namespaced names (`SetAnalyzerPlaybackSpeed`) just to dodge UE-engine-class shadow. Bad ergonomic tax that propagates through every BP call site forever.

## Workaround

Until fixed, two options:

1. **Rename the C++ UFUNCTION** to something unique against `UUserWidget` / `UActorComponent` / etc. ancestors of likely callers. Fragile — collision space depends on every future BP class that ever calls it.
2. **BP wrapper + `blueprint.graph.replace_node`** to manually pick the correct UFunction after BPIR emits the wrong one. Verbose, brittle to subsequent edits.

## Fix

Two changes inside the BPIR resolver:

1. **Reorder the cascade when `Target:` is explicit.** If the `call` has a `Target:` argument and `ResolveTargetClass` succeeds, run Step 4 (target-class lookup) **before** Step 1 (self-class). Self-class becomes fallback when the target class has no matching function. This mirrors what the UE BP node-creation flow does: the source pin's type determines the function picker's scope. Implementation lives in the cascade dispatcher in the compiler — the existing Step 4 logic is correct, only the ordering is wrong.

2. **Accept qualified `Class::Method` syntax** in `call` instructions as an explicit disambiguator. Parse `call ClassName::MethodName(...)` (also `pure`) by resolving `ClassName` via the existing class-resolution helpers used by `cast<>` and `subsystem<>`, then constraining the function search to that class's hierarchy. Decompiler should NOT emit qualified syntax by default — only when round-trip would otherwise be ambiguous. Documented as a v2 form in `bpir-language-reference.md`.

**Both fixes are required, not alternatives.** (1) is the silent-correctness fix — most call sites never need to think about which overload they got. (2) is the explicit-control fix — the user always has a way to pin a call to a specific class even when (1)'s heuristic disagrees with their intent, when the same name exists at multiple cascade levels for legitimate reasons, or when reading the BPIR makes the binding ambiguous on its face. Without (2), readers still can't tell from text alone which overload a call resolved to; without (1), users keep paying the wrapper-function tax for the common case. Land both.

## History
- `#1-initial-repro` `OPEN` reporter — Filed after `URaceAnalyzerSubsystem::SetPlaybackSpeed(float)` collided with `UUserWidget::SetPlaybackSpeed(UWidgetAnimation*, float)` in `W_AnalyzerControls`. Workaround: BP wrapper `ApplyPlaybackSpeed(float)` + `blueprint.graph.replace_node` to swap the inner node's UFunction to the subsystem's overload, then `connect_pins` to wire the `Speed` argument. Four-RPC dance for one call line.
- `#2-both-fixes-mandatory` `OPEN` reporter — Clarified the Fix section: (1) and (2) are both required, not "either alone is enough". (1) silently routes the common case; (2) gives explicit per-call control plus textual readability. Closing this issue requires both shipped.
- `#3-reorder-and-qualified-syntax` `IN-REVIEW` developer — Landed both fixes. Fix 1: `BpirCompiler.cpp` lifts the Step 4 (target-class) block to run before Step 1 (self-class) when `Target:` is present and resolves; falls through to the existing self/lib/broad cascade only on miss. Step 1b (SkeletonGeneratedClass) order preserved relative to Step 1. Fix 2: `BpirParser.cpp::ParseCallInstruction` splits `ClassName::MethodName` on the last `::`, stores the class in `Inst.TypeArg`; `BpirCompiler.cpp` short-circuits the cascade when `TypeArg` is non-empty for `call`/`pure`, resolving the class via `ResolveUClass` and the method on that class only (no fall-through on miss — explicit `Class::` is an explicit demand). Decompiler `BpirTextEmitter.cpp::GetFunctionDisplayName` emits `OwningClass::FuncName` when the unqualified cascade would resolve to a different `UFunction*` than the bound one; `EmitCallNode`/`EmitLatent`/`EmitPure` callsites skip outer `FormatNameToken` wrapping for qualified output (each half formats independently). Docs updated in `bpir-language-reference.md` (new "Qualified `ClassName::MethodName` syntax" subsection under §2.1) and `bpir-compiler-internals.md` (new Step 0 + Step 4-first preamble; Step 4 narrative re-framed as fallback position). Regression tests `FBpirCallTargetClassWinsOverSelfClassShadowTest`, `FBpirCallQualifiedClassMethodSyntaxTest`, and `FBpirCallQualifiedSyntaxUnresolvedClassFailsTest` in new file `Tests/Bpir/TestBpirTargetClassPriority.cpp` with shared test-only `UCLASS` fixture `TestBpirTargetClassPriorityFixture.h` (`UBpirTargetClassPriorityTestSubsystem::SetPlaybackSpeed(float)` collides with `UUserWidget::SetPlaybackSpeed`).
- `#4-verify-regression-tests-pass` `DONE` tester — Verified: ran the three new regression tests via `system.run_tests` (tests: `EditorAutomationRpcGateway.bpir.compiler.target_class_priority.{ShadowedByUUserWidget, QualifiedClassMethodSyntax, QualifiedSyntaxUnresolvedClassFails}`). Job `j_20260521T043944_4eaa21f2` completed in ~5s with `has_errors: false`, all three `resolvedTests`, empty `missingTests`. Both fixes (target-class-wins reorder and qualified `Class::Method` syntax incl. unresolved-class failure path) confirmed live.
- `#5-behavioral-verify-qualified-syntax` `DONE` tester — Behavioral verification on temp Widget BP `/Game/App/UI/Test/W_McpVerifyTemp_bpir_target_shadow`. `blueprint.compile_bpir` with `call NonExistentClass::SomeMethod()` returned `COMPILE_FAILED` with message `Line 2: Unresolved class 'NonExistentClass' in qualified call 'NonExistentClass::SomeMethod'` — matches Fix 2's unresolved-class branch. `blueprint.compile_bpir` with `call KismetSystemLibrary::PrintString(InString: "hello")` returned `compiled: true, nodeCount: 1, errors: []` — qualified `Class::Method` syntax resolves and emits a node. Temp BP deleted (`asset.delete` → `deletedCount: 1, existsAfter: false`). Frontmatter `status:` corrected from `IN-REVIEW` to `DONE` (was inconsistent with #4's DONE label).
