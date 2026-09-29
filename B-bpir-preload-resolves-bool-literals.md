---
id: B-bpir-preload-resolves-bool-literals
title: "BPIR PreloadExternalClasses sends boolean and other bare literals to ResolveUClass, logging \"Failed to find object 'Class true'\" across ~70 tests"
status: OPEN
severity: Low
category: bug
tags: [bpir, compiler, log-noise, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T09:01:03Z
---

# BPIR preload pass tries to load `true` / `false` as classes

`FBpirCompiler::PreloadExternalClasses` (`Source/PinWright/Private/Compiler/BpirCompiler.cpp` ~4516)
collects every instruction `TypeArg` and `Arg.Value` as a class-path candidate and calls
`ResolveUClass` on each one. Its `AddCandidate` filter rejects locals, `$` refs, enum literals,
quoted strings, struct literals, composite expressions and pure numerics, but not boolean literals
or other bare words. Each `true` / `false` / `nullptr` argument therefore becomes a `LoadObject`
attempt and a `LogUObjectGlobals: Warning: Failed to find object 'Class true'` line.

Measured on the gap-quick-wins verification run (host log `Saved/Logs/pw_gapwave_groups.log`):
86 × `Class true`, 8 × `Class false`, 1 × `Class nullptr`, 1 × `Class maybe`, emitted from 73
distinct tests, for example `bpir.round_trip.While`, `bpir.round_trip.CustomEvent`,
`bpir.compiler.integration.ElseIfChain`, `bpir.decompiler.SelectNode` and
`bpir.split_input.VectorSelect`. The warnings are noise that looks like a class-resolution failure and sends log
readers after the wrong defect; each is also a wasted load attempt on the compile path.

The timeline opcode is already exempted (its args are template settings, not class paths); the
general literal case is not.

**Fix:** in `AddCandidate`, skip boolean literals (`true` / `false`, case-insensitive) and
`nullptr` / `None`, alongside the existing numeric-literal skip.

## History
- `#1-class-true-log-noise` `OPEN` reporter — Filed from the gap-quick-wins verification run: 96 bogus class-load warnings from 73 BPIR/core tests, traced to `PreloadExternalClasses` feeding bare literals to `ResolveUClass`.
