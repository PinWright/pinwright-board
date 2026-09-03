---
id: B-bpir-get-property-into-register-unresolved
title: "BPIR `%r = get $Component.Property` compiles but the register is unusable on the next line ('Could not resolve value'), while `bpir.instructions` documents that exact form as valid"
status: OPEN
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
