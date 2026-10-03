---
id: B-bpir-bare-class-arg-logs-unknown-value-warning
title: "BPIR bare class-name argument (`ActorClass: ACharacter`) compiles correctly but logs a spurious `Unknown value reference format` warning"
status: OPEN
severity: Low
category: bug
tags: [bpir, compile_bpir, class-pin, log-noise, value-resolver]
encounters: 1
lastSeen: 2026-10-02T20:00:00Z
---

# Bare class-name arguments log a warning on the path that accepts them

`call GetAllActorsOfClass(WorldContextObject: self, ActorClass: ACharacter)` compiles and
sets `ActorClass` to `/Script/Engine.Character` (`OutActors` is typed as an array of
Character), yet each compile logs
`LogBpirValueResolver: Warning: Unknown value reference format: 'ACharacter'`.

`WireDataPins` (`Compiler/BpirCompiler.cpp`) first calls `FBpirValueResolver::ResolveValue`,
whose bare-identifier fall-through (`BpirValueResolver.cpp`, end of `ResolveValue`) logs
that warning and returns null; only afterwards does `WireDataPins` try
`ResolveUClass(Arg.Value)` and apply the class as the pin default. The bare form is in
use across the suite (`ActorClass: AActor` in several BPIR tests), so the warning is
pure noise: it misleads anyone reading a log for the cause of a failure, and under
`bElevateLogWarningsToErrors` it would fail a test that compiled correctly.

## Repro

Automation log of run1 for `PinWright.bpir.compiler.array_wildcard.DynamicOutputArrayMismatchNamesTypes`:
two `Unknown value reference format: 'ACharacter'` warnings, while the test's
"OutActors is typed as an array of Character" precondition passed.

## Expected

The bare-identifier warning fires only when nothing (register, parameter, class) resolves
the value, e.g. log it from `WireDataPins` after the class fallback fails.

## History

- `#1-filed` `OPEN` reporter — Seen in run1's automation log while diagnosing G03's array_wildcard failures (those failed on an unrelated assertion); the class literal resolved correctly despite the warning.
