---
id: B-bpir-forward-call-loses-parameter-pins
title: "compile_bpir forward call to a function declared later in the same code block loses every parameter pin — 'Could not find target pin X ... Available pins: self'"
status: DONE
severity: Medium
category: bug
tags: [bpir, compile_bpir, forward-reference, function, parameters, phase-1-5, skeleton]
encounters: 1
lastSeen: 2026-09-02T21:58:00Z
---

# A forward `call` to a same-block function sees only `self`

## Repro

One `compile_bpir` call on `/Game/FPS/UI/WBP_PauseMenu` declaring a function and calling it from a
`Tick` in the same code string:

```
entry function ApplyFocusVisual(object<Widget> Mark, object<TextBlock> Label, bool bFocused) {
    ...
    return
}

entry event Tick(struct<Geometry> MyGeometry, float InDeltaTime) {
    %f0 = call HasKeyboardFocus(Target: $ResumeButton)
    call ApplyFocusVisual(Mark: $ResumeMark, Label: $ResumeLabel, bFocused: %f0)
    ...
}
```

```
-> [COMPILE_FAILED]
   Line 14: Could not find target pin 'Mark' on node 'ApplyFocusVisual' Available pins: self;
   Line 14: Could not find target pin 'Label' on node 'ApplyFocusVisual' Available pins: self;
   Line 14: Could not find target pin 'bFocused' on node 'ApplyFocusVisual' Available pins: self;
   (repeated for lines 16 and 18)
```

Splitting into **two** `compile_bpir` calls — the `entry function` alone first, then the `Tick`
alone — compiles both cleanly with identical text. So the function and its signature are fine; only
the same-call forward reference is broken.

## Diagnosis

The node for `ApplyFocusVisual` is created, so name resolution succeeded; it is created with **no
parameter pins**, only the `self` pin. That is what a `UK2Node_CallFunction` looks like when it
resolves against a `UFunction` whose parameters have not been generated yet — i.e. the callee exists
on the skeleton class as a stub at the moment the caller is emitted.

`bpir.entry-points` § "Cross-entry calls and forward references" states: *"Phase 1 creates all entry
points, Phase 1.5 regenerates the Blueprint skeleton so they are resolvable as UFunctions on
`SkeletonGeneratedClass`, and Phase 2 emits the bodies; a forward `call MyFunc()` on line 2 can
therefore target a function declared on line 50."* The zero-argument example in that sentence is
exactly the case that works. With arguments it does not: the skeleton regeneration evidently
publishes the function symbol before its parameter list, or `AllocateDefaultPins` runs against the
pre-regeneration signature.

## What should happen

Phase 1.5 must regenerate the skeleton far enough that a forward-declared function's **parameters**
are present before Phase 2 allocates pins on its call sites — the documented promise. Failing that,
the doc must say forward references only work for zero-argument callees, and the error should name
the real cause ("callee signature not yet generated; declare it in an earlier compile_bpir call")
rather than reporting the caller's own argument names as unknown pins, which sends you looking for a
typo in your own code.

**Workaround:** author a function in its own `compile_bpir` call before any call site that passes
arguments to it.

severity rationale: impact=soft blocker with a clean workaround, but the diagnostic actively
misdirects (it blames the caller's arguments) x reach=any multi-entry BPIR block that factors shared
logic into a helper with parameters, which is normal structure -> Medium.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) adding a focus-highlight helper to `/Game/FPS/UI/WBP_PauseMenu` while fixing critic defect 5. One `compile_bpir` containing `entry function ApplyFocusVisual(object<Widget> Mark, object<TextBlock> Label, bool bFocused)` plus an `entry event Tick` calling it three times failed with `Could not find target pin 'Mark'/'Label'/'bFocused' ... Available pins: self` on every call site. The identical text split across two sequential `compile_bpir` calls (function first, then Tick) compiled clean both times, and `blueprint.compile` reports `UpToDate`. The `Available pins: self` detail is the evidence that the callee node was built against a parameterless stub. Contrast with `bpir.entry-points` § "Cross-entry calls and forward references", which promises this works via the Phase 1.5 skeleton regeneration; the promise holds for zero-arg forward calls only.
- `#2-skeleton-recompile-on-created-function` `IN-REVIEW` developer — Reproducible in current source; root cause found. `SetupFunction` calls `FBlueprintEditorUtils::AddFunctionGraph`, which runs `MarkBlueprintAsStructurallyModified` -> `RegenerateSkeletonOnly` **before** the parameter pins are added with `CreateUserDefinedPin`, so the skeleton gets a parameterless stub `UFunction`. Phase 1.5's `NeedsSkeletonRecompile` only checked `SGC->FindFunctionByName(graph name)`, found the stub, and skipped the recompile (unless a custom event was also created), so every same-compile call site allocated pins against the stub: `Available pins: self`. Zero-arg callees worked by accident, which is why the doc example held. The condition is not forward-reference-specific: any same-compile call with arguments to a function created in that compile hit it. Fix: `NeedsSkeletonRecompile` now returns true whenever Phase 1 created a function graph (`CreatedFunctionGraphs.Num() > 0`) in `Source/PinWright/Private/Compiler/BpirCompiler.cpp`. Test `PinWright.bpir.compiler.forward_reference.SameCompileCalleeKeepsParamPins` (`Tests/Bpir/TestBpirForwardReference.cpp`): a function and an event both call `ParamHelper(int Count, bool bFlag)` declared last in the same compile; asserts success, both call nodes carry `Count`/`bFlag`, and each keeps its literal. Docs: `docs/wiki-src/bpir.entry-points.md` (forward call with arguments), `docs/bpir-compiler-internals.md` Phase 1.5, `docs/bpir-test-matrix.md`, CHANGELOG. Compile-checked with UBT -SingleFile; not yet run.
- `#3-review-return-pin` `IN-REVIEW` developer — Review follow-up: `PinWright.bpir.compiler.forward_reference.SameCompileCalleeKeepsParamPins` now declares the later callee `-> int`, binds `%r = call ParamHelper(...)` in the caller, and asserts each call site carries an int `ReturnValue` output pin as well as the parameters. Syntax-checked with clang -fsyntax-only (fastcheck.sh, UBT module flags); no UHT-relevant declarations changed; not yet run.
- `#4-verified-linux` `DONE` tester — Run3 on PinWright 7230b41d, UE 5.8 Linux. `PinWright.bpir.compiler.forward_reference.SameCompileCalleeKeepsParamPins` passed non-skipped in run3/full: a function and an event both call `ParamHelper(int Count, bool bFlag) -> int`, declared last in the same compile_bpir, and every call site keeps `Count`/`bFlag`, its literal and an int `ReturnValue` pin. That is the reported `Available pins: self` case with arguments, fixed by forcing the Phase 1.5 skeleton recompile whenever Phase 1 created a function graph. The zero-arg guard `PinWright.bpir.compiler.forward_reference.FunctionCallsLaterFunction` also passed in run3/full. Doc: `docs/wiki-src/bpir.entry-points.md` now promises a forward `call MyFunc(Count: 3)` with its parameters in the same call, which matches the test. Limit: the reporter's WBP_PauseMenu asset was not replayed; the test uses a transient Blueprint.
