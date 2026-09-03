---
id: E-bpir-select-literal-options-stay-wildcard
title: "BPIR `select` with literal-only options never resolves its wildcard type, so an int/float select that is legal in the editor fails the whole compile with 'The type of Option 0 is undetermined'"
status: OPEN
severity: Medium
category: enhancement
tags: [bpir, compile_bpir, select, wildcard, type-inference, error-message]
encounters: 1
lastSeen: 2026-09-03T02:45:00Z
---

# BPIR `select` cannot take literal options

## Symptom

A `select` whose `true` / `false` options are literals rather than wired values leaves the
`UK2Node_Select` pins as wildcards, and the whole-Blueprint compile fails:

```
The type of  Option 0  is undetermined.  Connect something to  Select  to imply a specific type.
The type of  Option 1  is undetermined.  Connect something to  Select  to imply a specific type.
The type of  Return Value  is undetermined. ...
Default value '3' for  Option 1  is invalid: 'Unsupported type Wildcard on pin Value'
```

## Repro

```
entry override Tick(struct<Geometry> MyGeometry, float InDeltaTime) {
    %f2 = call HasKeyboardFocus(Target: $QuitButton)
    %f3 = call HasKeyboardFocus(Target: $BackButton)
    %i3 = select(cond: %f3, true: 3, false: -1)
    %i2 = select(cond: %f2, true: 2, false: %i3)
    ...
}
```

-> `BLUEPRINT_COMPILE_FAILED`, four Select nodes reported, compile rolled back.

Note the inconsistency that makes this surprising: a `select` whose options are **struct or enum
literals** works today and is emitted by the decompiler —

```
%n7: struct<LinearColor> = select(Index: %n6, false: %n4.LinearColor, true: %n5.LinearColor)
%n1: enum<ESlateVisibility> = select(Index: %n0, false: ESlateVisibility::Collapsed, true: ESlateVisibility::HitTestInvisible)
```

so `select` reads as generally usable, and it is only the numeric-literal case that dies. In the
editor, typing `2` and `3` into a Select node's option pins resolves the wildcard immediately; the
gap is that BPIR never performs that resolution.

## Workaround

Compute the value arithmetically and avoid `select` entirely. For a "which one of N booleans is
true" index:

```
%n0 = call Conv_BoolToInt(InBool: %f0)
%n1 = call Conv_BoolToInt(InBool: %f1)
%w1 = call Multiply_IntInt(A: %n1, B: 2)
%s1 = call Add_IntInt(A: %n0, B: %w1)
%idx = call Subtract_IntInt(A: %s1, B: 1)      # -1 when none are set
```

Correct, but four nodes where one Select would do, and it obscures the intent.

## Suggested fix

When every option on a `select` is a literal, infer the pin type from the literals (all options
must agree) and call the same pin-type propagation the editor uses, before the Blueprint compile
runs. Failing that, let the author state it — `select<int>(cond: ..., true: 2, false: -1)` — and
say so in the error, which currently names the symptom but not the remedy.

## Related

- `E-bpir-cast-failure-pin-is-named-fail` — the same shape of problem: an error that names what is
  wrong without naming what would be right.
