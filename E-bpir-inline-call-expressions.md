---
id: E-bpir-inline-call-expressions
title: "BPIR doesn't allow inline `call ...` as argument values; forces verbose %ref hoisting"
status: WONTFIX
severity: Low
category: ergonomic
tags: [bpir, parser, value-resolver, ergonomic, call, select, branch]
---

# Inline `call FuncName(...)` not accepted in argument position

BPIR `select`, `branch`, and the args of an outer `call` accept `$var`,
`%ref`, `cast<T>(...)`, and literal values as argument expressions, but
not nested `call FuncName(...)`. Every intermediate function call must
be hoisted to its own `%nN: T = call FuncName(...)` line before its
result can be passed as an argument. For dense logic with many small
comparisons (e.g. building per-button selection state across 10 lap
buttons) the resulting BPIR is ~3× longer than the equivalent
imperative form would be.

The parser already understands `call` as an expression head on the RHS
of `%nN: T = call ...` (see `ParseCallInstruction` in
`Source/EditorAutomationRpcGateway/Private/Compiler/BpirParser.cpp:1475`);
allowing the same head to nest inside argument slots — recognised by
`FBpirValueResolver::ResolveValue` in
`Source/EditorAutomationRpcGateway/Private/Compiler/BpirValueResolver.cpp:140`
(which today branches only on `%`, `$`, `self`, `cast<`, and literal,
then errors out with "Could not resolve value '...' for pin '...'") —
would let authors write
`select(Index: call EqualEqual_IntInt(A: $LapIdx, B: 0), false: 0.4, true: 1.0)`
directly.

This is an ergonomic improvement, not a correctness bug — the current
behaviour is documented and the workaround always succeeds.

## Repro

```
%s1: float = select(Index: call EqualEqual_IntInt(A: $LapIdx, B: 0), false: 0.400000, true: 1.000000)
```

`blueprint.compile_bpir` returns
`Could not resolve value 'call EqualEqual_IntInt(A: $LapIdx, B: 0)' for pin 'Index'`.

## Workaround

Hoist each call into its own line:

```
%e1: bool = call EqualEqual_IntInt(A: $LapIdx, B: 0)
%s1: float = select(Index: %e1, false: 0.400000, true: 1.000000)
```

Works, but every per-button branch doubles in length. 10 buttons = 30
extra lines instead of 10.

## Proposed fix

Extend `FBpirValueResolver::ResolveValue` with a `call`-headed branch
that emits a pure `K2Node_CallFunction` (mirroring the existing
`cast<T>(...)` inline branch which emits a pure `K2Node_DynamicCast`)
and returns its primary output pin. The parser's `ParseArgs` already
preserves the argument string verbatim through balanced parens, so the
recursive parse can be done lazily inside the resolver.

Restrict to functions marked `BlueprintPure` to avoid implicit
exec-side effects from a syntactically-pure inline expression.
`PreEmitVariableRefs` would also need to recurse into the inner call's
arguments to pre-emit any `$param` references they reference (same
pattern used for the cast-RHS fix in `E-bpir-cast-as-set-rhs`).

## History
- `#1-initial-repro` `OPEN` reporter — Authoring per-lap-button select expressions (10 buttons × 1 call each) required hoisting every `EqualEqual_IntInt` to its own `%eN: bool = call ...` line before referencing it from `select(Index: %eN, ...)`. 30 lines for what could be 10. Parser already understands `call` as expression head on RHS of `%nN: T = call ...` (BpirParser.cpp:1475); `FBpirValueResolver::ResolveValue` (BpirValueResolver.cpp:140) only handles `%`, `$`, `self`, `cast<`, literal — no `call`-headed branch, so falls through to "Could not resolve value 'call EqualEqual_IntInt(...)' for pin 'Index'". Closest neighbour `E-bpir-cast-as-set-rhs` (DONE) added the same shape for `cast<T>(...)`.
- `#2-wontfix-low-roi` `WONTFIX` developer — Low severity, ergonomic-only; the `%tmp = call F(...)` hoist workaround is already in standard use and always succeeds. The parallel DONE sibling `E-bpir-cast-as-set-rhs` (same shape, for `cast<T>(...)` on the RHS) required five review rounds to land and produced a wildcard-survival regression in round #3 that had to be patched. Inline `call`-headed expressions would add the same parser/resolver complexity (recursive arg parse, `BlueprintPure` gate, `PreEmitVariableRefs` recursion into nested call args) and history shows that surface is hard to land cleanly. Closing without code change; reopen if a concrete authoring task hits >100 hoisted intermediates and the workaround becomes the bottleneck.
