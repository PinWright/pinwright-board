---
id: B-bpir-get-property-into-register-unresolved
title: "BPIR `%r = get $Component.Property` compiles but the register is unusable on the next line ('Could not resolve value'), while `bpir.instructions` documents that exact form as valid"
status: DONE
severity: Medium
category: bug
tags: [bpir, compile_bpir, get, property-read, register, documentation-mismatch]
encounters: 1
lastSeen: 2026-09-03T03:53:00Z
---

# `get` into a register produces a register nothing can read

## Repro

UE 5.8, EAContentExamples58, PinWright on 27145, `blueprint.compile_bpir` on
`/Game/FPS/Player/BP_FPSCharacter` (an `ACharacter` subclass with a `UCameraComponent` named
`Camera`).

**Fails:**

```
entry function UpdateViewmodelFOV() {
    %fov = get $Camera.FieldOfView
    %halfMain: double = call Multiply_DoubleDouble(A: %fov, B: 0.5)
    ...
}
-> [COMPILE_FAILED] Line 3: Could not resolve value '%fov' for pin 'A'
```

The `get` line itself raises nothing. The error lands on the **next** line, on the consumer.

**Works** — the same read, inline in value position:

```
%halfMain: double = call Multiply_DoubleDouble(A: $Camera.FieldOfView, B: 0.5)
```

Also fails, differently, which is how I found the first form:

```
%fov: double = $Camera.FieldOfView
-> [COMPILE_FAILED] Line 2: Unknown instruction keyword after '=': $Camera.FieldOfView
```

## Against the documentation

`bpir.instructions` § "External property access" lists, as valid:

```
%h = get $PlayerState.Stats.Health
```

which is the same shape as the failing line (a `$name.prop` chain read into a `%` register). So
either the doc is describing a form the parser does not implement, or the parser emits the
`VariableGet` without registering its output pin under the register name. Given that the identical
chain resolves perfectly in value position, the read itself is fine — it is the **binding of the
result to `%fov`** that does not happen.

## Why it matters

Inline-only property reads are a real constraint, not a style preference: a value used twice has to
be re-read twice (two `VariableGet` nodes for one property), and a read whose consumer is a
`branch` condition or a `make<>` field cannot always be expressed inline at all. It also makes
`blueprint.decompile` output non-round-trippable in the general case, since the decompiler is free
to emit whichever form matches the graph.

## Asked for

Either make `%r = get $obj.Prop` bind the register — the documented behaviour — or remove that line
from `bpir.instructions` and say plainly that external property reads are value-position only. The
silent version is the expensive one: the failure surfaces one line later, on an unrelated pin, and
reads like a typo in the consumer.

## Related

`B-bpir-member-call-through-chained-property` — same family (BPIR property-ref resolution), different
symptom: there a chained property fails as a member-call `Target:`; here it fails to bind a register.
Worth fixing together if the resolution path is shared.

severity rationale: impact=a documented form does not work and fails with a misleading, misplaced
error x reach=any BPIR body that reads a component or sub-object property more than once
-> Medium

## History
- `#1-filed` `OPEN` reporter — Hit while authoring `UpdateViewmodelFOV` on the FPS PLAYER stream (read `Camera.FieldOfView`, halve it, take `DegTan`). Worked around by inlining `$Camera.FieldOfView` directly into the `Multiply_DoubleDouble` call, which compiled first time; the function is on disk and byte-verified. Both failing forms and the working one are quoted above verbatim from the RPC responses.
- `#2-get-dollar-as-alias` `IN-REVIEW` developer — Reproducible in current source. The parser's `get` handler stored the operand verbatim as a self-member variable name, so `%fov = get $Camera.FieldOfView` emitted a `UK2Node_VariableGet` for a variable literally named `$Camera.FieldOfView`: no output pin, `PrimaryOutputPin` null, and the consumer on the next line failed `Could not resolve value '%fov'`. Fix: `Source/PinWright/Private/Compiler/BpirParser.cpp` — a `get` operand starting with `$` now parses as an `Alias` of that value reference (the same resolution the inline value-position form uses); and `PreEmitVariableRefs` in `Source/PinWright/Private/Compiler/BpirCompiler.cpp` pre-emits an alias RHS through the shared value-ref path, so a dotted `$obj.Prop` RHS gets its external VariableGet (it used to call `PreEmitDollarVar` on the whole string). Plain `get MyVar` is unchanged. Not changed: `%fov = $Camera.FieldOfView` (alias with a dot suffix) is still rejected — that is the deliberate parser rule pinned by `PinWright.bpir.parser.AliasDotSuffixRejected`; `get` is the documented spelling. Test `PinWright.bpir.compiler.chained_property.GetDollarChainBindsRegister` (`Tests/Bpir/TestBpirChainedPropertyRefs.cpp`): `%fov = get $Cam.FieldOfView` on an `object<CameraComponent>` member feeding `Multiply_DoubleDouble(A: %fov)`; asserts success, no VariableGet named `$Cam.FieldOfView`, and the external `FieldOfView` VariableGet wired into `A`. Docs: `docs/wiki-src/bpir.instructions.md` §2.2, `docs/bpir-test-matrix.md`, CHANGELOG. Compile-checked with UBT -SingleFile; not yet run.
- `#3-review-nits` `IN-REVIEW` developer — Review follow-up. A `get $...` register now also works as a member-call `Target:` (alias class resolution added in `ResolveViaTargetArg`, see `B-bpir-member-call-through-chained-property` #3; test `PinWright.bpir.compiler.chained_property.MemberCallTargetThroughGetAlias`). `%r = $obj.Prop` (no `get`) is still rejected, but now with `Property access cannot be assigned directly: '$obj.Prop'. Use '%r = get $obj.Prop'` instead of `Unknown instruction keyword after '='` (`Source/PinWright/Private/Compiler/BpirParser.cpp`, `$`-ref dot-suffix branch); `PinWright.bpir.parser.AliasDotSuffixRejected` now also asserts the hint. `bpir.instructions.md` §2.2 states the `get $...` form takes no `@(x, y)` placement and no trailing exec targets. Syntax-checked with clang -fsyntax-only (fastcheck.sh, UBT module flags); no UHT-relevant declarations changed; not yet run.
- `#4-verified-linux` `DONE` tester — Run3 on PinWright 7230b41d, UE 5.8 Linux, all non-skipped in run3/full. `PinWright.bpir.compiler.chained_property.GetDollarChainBindsRegister`: `%fov = get $Cam.FieldOfView` feeding `Multiply_DoubleDouble(A: %fov)` compiles, with the external `FieldOfView` VariableGet wired into `A` and no bogus `$Cam.FieldOfView` variable node. That is the documented form now binding its register (ask option 1). `PinWright.bpir.compiler.chained_property.MemberCallTargetThroughGetAlias`: the register also works as a member-call `Target:`. `PinWright.bpir.parser.AliasDotSuffixRejected`: `%r = $obj.Prop` is still rejected, now with the `Use '%r = get $obj.Prop'` hint instead of `Unknown instruction keyword`. Doc: `docs/wiki-src/bpir.instructions.md` §2.2 describes the `get $...` form, its no-placement rule and the hint. Limit: the reporter's BP_FPSCharacter asset was not replayed; the tests use transient Blueprints.
