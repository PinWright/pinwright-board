---
id: B-bpir-multigate-native-node
title: "BPIR `macro MultiGate` never compiles on any UE 5.x, and a native MultiGate decompiles as `sequence`"
status: IN-REVIEW
severity: Medium
category: bug
tags: [bpir, compiler, decompiler, macro, multigate, test-skip, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T16:51:00Z
---

# BPIR treats MultiGate as a StandardMacros graph; the engine ships it as the native `UK2Node_MultiGate`

MultiGate is not a macro in any supported engine. `/Engine/EditorBlueprintResources/StandardMacros.uasset`
is the same 422,905-byte file on 5.3-5.8 and its name table holds `DoOnce`, `FlipFlop` and `Gate`
but no `MultiGate`. The engine node is `UK2Node_MultiGate : UK2Node_ExecutionSequence`
(`Engine/Source/Editor/BlueprintGraph/Classes/K2Node_MultiGate.h`, `UCLASS(MinimalAPI)`, present
on 5.3 and 5.8), with pins `execute`, `Reset`, `IsRandom`, `Loop`, `StartIndex` and `Out 0..N`.

Two product defects followed:

- **Compile:** the `Macro` opcode (`Compiler/BpirCompiler.cpp` ~6564) only called
  `FCodeNodeEmitter::CreateMacroNode`, which searches StandardMacros and the target Blueprint's
  macro graphs. The documented `macro MultiGate(IsRandom: ..., Loop: ...) [0 -> @a, ...]`
  (`docs/wiki-src/bpir.instructions.md` §2.4, `bpir.examples.multigate.md`) therefore failed on
  every engine with `Macro 'MultiGate' not found in StandardMacros or target Blueprint MacroGraphs`.
- **Decompile:** `GraphWalker` classifies by the most-derived registered class, and only
  `UK2Node_ExecutionSequence` was registered, so a native MultiGate in a real Blueprint was emitted
  as `sequence(N)`. Recompiling that text produced a plain ExecutionSequence: silent loss of the
  gate semantics (Reset, IsRandom, Loop, StartIndex, one-output-per-fire) on decompile -> recompile.

The three tests covering the form (`bpir.compiler.integration.MacroMultiGate`,
`bpir.decompiler.MacroMultiGate`, `bpir.round_trip.MacroMultiGate`) matched the "not found" error
and emitted `PINWRIGHT_ASSERTIONS_SKIPPED reason=multigate-macro-unavailable` under the false premise
that MultiGate is absent from StandardMacros only on "UE 5.6+". They measured nothing on any engine
(host log `Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log` lines 20530,
21770, 23531).

**Fix:** implemented, see History.

**Output count:** exec targets list only connected outputs, so the count travels as the
`outputs: N` node setting (see `#3-outputs-count-arg`); unconnected trailing `Out N` pins survive
a round trip.

## History
- `#1-multigate-never-compiles` `OPEN` reporter — Full offscreen suite on UE 5.8 skipped `bpir.compiler.integration.MacroMultiGate`, `bpir.decompiler.MacroMultiGate` and `bpir.round_trip.MacroMultiGate` with `multigate-macro-unavailable`. Root cause: StandardMacros has no MultiGate graph on 5.3-5.8 (identical asset, name table has only DoOnce/FlipFlop/Gate); MultiGate is the native `UK2Node_MultiGate`. BPIR's `Macro` opcode only looks up macro graphs, so `macro MultiGate` fails everywhere, and `GraphWalker` classifies a native MultiGate through its `UK2Node_ExecutionSequence` base, decompiling it as `sequence(N)`.
- `#2-native-multigate-lane` `IN-REVIEW` developer — Compiler (`Compiler/BpirCompiler.cpp`, `Macro` case of the emit switch): when the name is `MultiGate` and the target Blueprint has no same-named macro graph (user macros keep precedence), creates the native node via `CreateGenericK2Node("K2Node_MultiGate")`, grows `Out N` pins to cover the highest index an exec target names (`2`, `Out 2`, `Out2`), wires the incoming exec into `execute`, and leaves the cursor on the first output. `FindExecOutputPin`'s `Macro` case also maps a bare index `N` to `Out N`, so the documented `[0 -> @a]` form resolves. Decompiler: `Decompiler/GraphWalker.cpp` registers `UK2Node_MultiGate -> MacroInstance` (the most-derived walk picks it over Sequence); `Decompiler/BpirTextEmitter.cpp` `EmitMacro` accepts the native node and names it `MultiGate`, so it emits `%n = macro MultiGate(<non-default data pins>) [Out0 -> @out0, ...]`. Tests: the three skip branches are removed. The compiler test asserts one native `UK2Node_MultiGate`, zero MacroInstances, and both `Out 0` / `Out 1` wired; the decompiler test asserts `macro MultiGate(` present, no `sequence(`, both `Out0 ->` / `Out1 ->` targets; the round-trip test additionally asserts the recompiled graph holds exactly one native MultiGate and no other ExecutionSequence. All six .cpp files compile-checked with UBT `-SingleFile` on UE 5.8; tests not yet run.
- `#3-outputs-count-arg` `IN-REVIEW` developer — Removed the known limit that unconnected trailing outputs were dropped on a round trip. New node setting `outputs: N` on `macro MultiGate` (lowercase like Timeline's `length:`, because it names no pin; constants `BpirSharedConstants::MacroNames::MultiGate` / `MultiGateOutputsArg` in `Compiler/BpirSharedConstants.h`). Compiler (`Compiler/BpirCompiler.cpp`): the MultiGate lane validates `outputs` before creating the node (a non-positive or non-integer value is a compile error naming the argument), then grows `Out N` pins to max(N, highest wired index + 1); `ResolveTargetPinForOpcode` skips the `outputs` arg for `Macro` when the node is a native `UK2Node_MultiGate`, so `WireDataPins` does not look for an `outputs` pin. A user macro named MultiGate still receives `outputs` as an ordinary pin arg. Decompiler (`Decompiler/BpirTextEmitter.cpp` `EmitMacro`): always emits `outputs: <exec output count>` first for a native MultiGate, e.g. `macro MultiGate(outputs: 4) [Out0 -> @out0]`. New tests: `bpir.round_trip.MacroMultiGateKeepsUnwiredOutputs` (4 outputs, only Out 0 wired: compile, decompile carries `outputs: 4`, recompile keeps 4 outputs with only Out 0 wired) and `bpir.compiler.integration.MacroMultiGateOutputCount` (`outputs: 2` with `3 -> @last` grows to 4 and wires `Out 3`; `outputs: 0` fails naming the argument and leaves no MultiGate). Wiki: `docs/wiki-src/bpir.instructions.md` §2.4 and `docs/wiki-src/bpir.examples.multigate.md` document `outputs:` and the native-node mapping. `BpirCompiler.cpp`, `BpirTextEmitter.cpp`, `TestCompilerMacrosEntryPoints.cpp` and `TestBpirRoundTrip.cpp` compile-checked with UBT `-SingleFile` on UE 5.8; tests not yet run.
