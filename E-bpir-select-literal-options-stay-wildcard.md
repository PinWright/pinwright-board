---
id: E-bpir-select-literal-options-stay-wildcard
title: "BPIR `select` with literal-only options never resolves its wildcard type, so an int/float select that is legal in the editor fails the whole compile with 'The type of Option 0 is undetermined'"
status: DONE
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

## History
- `#1-post-wiring-literal-inference` `IN-REVIEW` developer — Reproduced by reading source, no editor run: `UK2Node_Select` types itself only when a typed pin connects (`K2Node_Select.cpp` `NotifyPinConnectionListChanged`); BPIR pre-typed a select only from a `%x: T =` annotation or an FText literal (`ResolveSelectResultPinType`), so a literal-only select, or a chain of them, stayed wildcard. Fixed with a post-wiring pass, `TypeUndeterminedSelects` in `Compiler/BpirCompiler.cpp`, run after Pass 3a in all three wiring loops: for every select still wildcard it takes the type of a typed linked pin (a neighbour select this pass just typed — fixed-point iteration, so chains resolve in any order), else infers from the option literals (decimal int, real if any is fractional, bool, plain quoted string; mixed kinds / nullptr / hex / enum / struct literals are not inferred), stamps it via `PreTypeSelectPins` and re-applies the raw literals so a `1.5f` lands as `1.5`. Running after wiring rather than at emit is deliberate: a select a typed consumer reaches keeps the consumer's type (`set FloatVar = select(..., true: 1, false: 0)` stays `real`, no conversion node), so nothing that compiles today changes. The annotation remedy (`%i: int = select(...)`) is now documented in `bpir.instructions.md` §2.3. Files: `Source/PinWright/Private/Compiler/BpirCompiler.cpp`, `docs/wiki-src/bpir.instructions.md`, `CHANGELOG.md`, `Source/PinWright/Private/Tests/Bpir/TestBpirWildcardResolution.cpp`. Test: `PinWright.bpir.compiler.select_literal.OptionsResolveType` (the ticket's `%i3`/`%i2` chain resolves to int with `-1`/`3`/`2` defaults, int+real -> real with `1.5f` -> `1.5`, bools -> bool, a real consumer keeps real with no `Conv_IntToDouble`, then a Kismet compile is not `BS_Error`; the pin-type assertions fail before the fix). Compile-checked with -SingleFile; not run.
- `#2-review-macro-wildcard-guard` `IN-REVIEW` developer — Review follow-up in `TypeUndeterminedSelects` (`Compiler/BpirCompiler.cpp`): literal inference is skipped when any pin of the select links to a wildcard pin on a non-Select node (e.g. a macro graph's wildcard tunnel via CompileBodyIntoGraph / InsertCodeAfterNode), so a legitimately generic select is not frozen to its literals' type; neighbour typing from a *typed* link still applies. A literal that cannot be re-applied once the pin is typed now adds a compile error at the select's line instead of being left silently on the pin.
- `#3-review-fixes` `IN-REVIEW` developer — Re-review follow-up (§4 #5): the `#2` guard (skip literal inference when a select links to a wildcard pin on a non-Select node) missed select chains in macro bodies: `%a = select(c,1,2)` -> `%b = select(d,%a,3)` -> wildcard output tunnel typed `%a` int from its literals, then `%b` int from `%a`. `TypeUndeterminedSelects` (`Compiler/BpirCompiler.cpp`) now drops `bLinksToGenericWildcard` and skips only the literal fallback when `UEdGraphSchema_K2::GetGraphType(SelectNode->GetGraph()) == GT_Macro`; typing from a typed neighbour still applies everywhere. New test `PinWright.bpir.compiler.select_literal.MacroSelectsStayWildcard` (macro with a two-select chain and a literal-only select into wildcard output tunnel pins, asserts all three Return Values stay wildcard; fails with the old guard). Wiki §2.5 select paragraph (`docs/wiki-src/bpir.instructions.md`) and the CHANGELOG entry state the macro exception. fastcheck OK; not run in an editor.
- `#4-verified-linux` `DONE` tester — Run3 on PinWright 7230b41d, UE 5.8 Linux, both non-skipped in run3/full. `PinWright.bpir.compiler.select_literal.OptionsResolveType`: the ticket's own `%i3`/`%i2` literal select chain resolves to int with defaults -1/3/2. Int+real infers real (`1.5f` -> 1.5), bools infer bool, a real consumer keeps real with no Conv_IntToDouble, and the Kismet compile is not BS_Error. That meets the suggested fix: infer the pin type from the literals. `PinWright.bpir.compiler.select_literal.MacroSelectsStayWildcard`: selects in macro graphs stay wildcard as intended. Doc: `bpir.instructions.md` documents the inference, the `%i: int = select(...)` annotation remedy and the macro exception. Limit: mixed-kind/hex/enum/struct literal options are not inferred, and the annotation is the documented route for those.
