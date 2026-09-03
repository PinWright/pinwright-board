---
id: B-bpir-forward-call-loses-parameter-pins
title: "compile_bpir forward call to a function declared later in the same code block loses every parameter pin — 'Could not find target pin X ... Available pins: self'"
status: OPEN
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
