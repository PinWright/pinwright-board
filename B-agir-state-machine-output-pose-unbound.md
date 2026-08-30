---
id: B-agir-state-machine-output-pose-unbound
title: "AGIR round-trip break: decompiled `output %state_machine_0` never binds — compile rejects its own decompiler output for single-state-machine AnimBPs"
status: IN-REVIEW
severity: High
category: bug
tags: [agir, animgraph, roundtrip, state-machine, decompile, compile]
---

# AGIR decompile → compile round-trip is broken for a top-level state machine

`anim.decompile_agir` emits an AnimGraph whose `output` pose-link
references a `%state_machine_<n>` symbol, but it writes the state
machine itself with the **unassigned name-token block-opener** form
(`state_machine <Name> { … }`) — no `%state_machine_<n> = ` binding.
Feeding that **unmodified decompiler output** straight back into
`anim.compile_agir` fails:

```
[AGIR_SYMBOL_NOT_FOUND] Pose reference '%state_machine_0' did not resolve to a UAnimGraphNode_Base (line 3).
```

So the decompiler emits text its own compiler cannot consume. Any
AnimBP whose top-level `AnimGraph` output is a single state machine —
the overwhelmingly common shape for a locomotion AnimBP — cannot
round-trip. This blocks the canonical "decompile → edit AGIR text →
recompile" authoring workflow on the most ordinary state-machine asset.

## Why the existing AGIR cliff/state-alias tests didn't catch it

`F-agir-cliff-completion` and `F-agir-state-alias` made
`FAGIRRoundTripStateMachineTest` load-bearing — but only against the
**Lyra mannequin** `ABP_Mannequin_Base`. That fixture's `AnimGraph`
output flows through `save_cached_pose` / `use_cached_pose` and layered
blends, so its top-level `output` ref points at a cached-pose symbol,
not at `%state_machine_0` directly. The plain "AnimGraph output IS the
state machine" topology is never exercised by that fixture, so this
symbol-resolution gap slipped through.

## Root cause (source-confirmed)

The break is the direct consequence of `F-ir-grammar-harmonize` axis #3
(class-mnemonic register naming), whose history asserted *"Recompile is
name-blind (the compiler binds nodes by graph position, not by token
spelling), so the migration is zero-risk for round-trip identity."*
That assumption is false for the **state-machine output pose link**: the
compiler resolves `output %state_machine_0` **by symbol**, and the state
machine node is never registered under that symbol.

- Emitter — `AGIR/AGIRTextEmitter.cpp` `EmitStateMachine` (lines ~728-738):
  pre-allocates the AGIR-local id via `(void)IdFor(MachineNode)` so the
  downstream `output` ref reads `%state_machine_0`, but **deliberately
  does NOT write `%n = ` to the text** — it emits only the name-token
  opener `state_machine <name> {`. Its own comment says the parser "only
  accepts the unassigned `state_machine <name-token> {` opener form".
- Parser — `AGIR/AGIRParser.cpp` `ParseBlockHeader` stores the opener's
  name in `SymbolName`; `ResultName` stays empty for the name-token form
  (only the `%x = call …` assigned-call path sets `ResultName`).
- Compiler — `AGIR/AGIRCompiler.cpp` `EmitInstruction`, `case
  EAGIROpcode::StateMachine` (lines 788-813): reads `MachineName =
  Inst.SymbolName` and registers the machine node into `Symbols` **only
  if `Inst.ResultName` is non-empty** (`if (!Inst.ResultName.IsEmpty())
  Symbols.Add(Inst.ResultName, MachineNode);`). Because the decompiler
  never writes a `%state_machine_0 = ` binding, `ResultName` is empty,
  `Symbols.Add` is skipped, and the later `output %state_machine_0`
  pose-link reference resolves against nothing → `AGIR_SYMBOL_NOT_FOUND`.

The emitter pre-allocates a `%state_machine_0` id that is purely
emitter-internal — it appears in the `output` line but is never written
on the `state_machine` line and is never reconstructed by the
parser/compiler. The two halves of the round-trip disagree on how the
machine's output pose symbol is named/registered.

## What it should do

Pick one and make decompile + compile agree:
1. **Emitter writes the binding:** emit the assigned form
   `output %state_machine_0 = state_machine <name> { … }` (or otherwise
   surface the `%state_machine_0` token on the machine's own line) so the
   parser fills `ResultName` and the compiler registers it.
2. **Compiler binds the name-token machine to the consuming `output`
   ref:** when a `state_machine` block has an empty `ResultName`, register
   the created node under the pre-allocated mnemonic id the emitter used
   (i.e. teach the compiler the same `%state_machine_<n>` positional/order
   convention the emitter uses), so `output %state_machine_0` resolves.

Either way the invariant to restore: **`anim.decompile_agir` output fed
verbatim into `anim.compile_agir` must compile.** Add a round-trip
regression test against a *single-top-level-state-machine* fixture (e.g.
a synthetic AnimBP whose `AnimGraph` output is the state machine) so the
plain topology is load-bearing, not only the cached-pose-fronted Lyra
mannequin.

## Verbatim repro

1. `anim.decompile_agir` `{ assetPath: "/Game/ExampleContent/Animation_Basics/2-5_ABP_StateMachines" }`
   → emits (line 2-3):
   ```
   entry anim_graph AnimGraph {
       output %state_machine_0 guid="d0c56e7a-45c9-e0a3-137c-73881e25e0a0" @(0, 0)
       state_machine Run { guid="48c98cee-4454-f5e6-c057-c59fa4cd6078" @(-256, 0)
       ...
   }
   ```
   warnings: `[]`.
2. Feed that **unmodified** text back: `anim.compile_agir`
   `{ context: "<a duplicate of that AnimBP>", mode: "Replace", text: "<the decompiler output verbatim>" }`
   → error:
   ```
   [AGIR_SYMBOL_NOT_FOUND] Pose reference '%state_machine_0' did not resolve to a UAnimGraphNode_Base (line 3).
   ```
   Default (Extend) mode fails identically. Replayed via
   `mcp__editor-automation__call` against a duplicate
   (`2-5_ABP_StateMachines_AGIRReplay`) to avoid mutating the original.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed against
  `/Game/ExampleContent/Animation_Basics/2-5_ABP_StateMachines` (Content
  Examples). `anim.decompile_agir` emits `output %state_machine_0` plus a
  name-token `state_machine Run {` opener with no `%state_machine_0 = `
  binding; feeding the verbatim output back into `anim.compile_agir`
  (Replace and default Extend) fails with `[AGIR_SYMBOL_NOT_FOUND] Pose
  reference '%state_machine_0' did not resolve to a UAnimGraphNode_Base
  (line 3)`. Root-caused in source: `AGIRTextEmitter.cpp:EmitStateMachine`
  pre-allocates the `%state_machine_0` LocalId for the `output` ref but
  intentionally omits the `%n = ` text; `AGIRParser.cpp:ParseBlockHeader`
  leaves `ResultName` empty for the name-token opener; `AGIRCompiler.cpp`
  `case StateMachine` (lines 788-813) only does `Symbols.Add` when
  `ResultName` is non-empty, so the consuming `output %state_machine_0`
  ref resolves against nothing. Direct falsification of
  `F-ir-grammar-harmonize` axis #3's "recompile is name-blind / zero-risk
  for round-trip identity" claim. Existing `FAGIRRoundTripStateMachineTest`
  passes only because the Lyra mannequin fixture fronts its AnimGraph
  output with a cached pose, never the bare single-state-machine topology.
- `#2-register-name-token-state-machine` `IN-REVIEW` developer — Fixed via
  remedy #2 (compiler registers the name-token machine under the emitter's
  pre-allocated positional id). In `AGIRCompiler.cpp` the top-level driver
  `CompileBlockIntoGraph` now owns a block-scoped `TMap<FString,int32>
  MnemonicCounters` mirroring the decompiler's `IdFor()` per-mnemonic counter,
  threaded into `EmitInstruction`; the `case EAGIROpcode::StateMachine` advances
  the `state_machine` counter once per state_machine instruction (in instruction
  = emit order) and, when `Inst.ResultName` is empty (the name-token opener
  form), registers the created node into `Symbols` under the same positional id
  `state_machine_<n>` the emitter referenced from `output %state_machine_<n>`.
  An explicit `%n = ` binding still wins when present. This makes the consuming
  `output %state_machine_0` wire resolve in `ResolvePendingPoseWires` instead of
  failing with `AGIR_SYMBOL_NOT_FOUND`. No parser/emitter change needed (remedy
  #2 is the narrower fix; remedy #1 would have required teaching
  `ParseAssignedCall` to accept an assigned `state_machine` opcode). Files:
  `Source/EditorAutomationRpcGateway/Private/AGIR/AGIRCompiler.cpp`. Regression
  test: `FAGIRRoundTripTopLevelStateMachineOutputTest`
  (`EditorAutomationRpcGateway.anim.agir.RoundTripTopLevelStateMachineOutput`)
  in `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestAnimGraphHandlers.cpp`
  — authors a fresh AnimBP whose AnimGraph output is wired directly to a single
  top-level state machine (the bare topology the Lyra fixture never exercises),
  asserts the decompiler emits `output %state_machine_0`, then compiles that
  verbatim text into a fresh target and asserts success plus the Result pin is
  wired; fails with `AGIR_SYMBOL_NOT_FOUND` if the fix is reverted. Roots in a
  regression of `F-ir-grammar-harmonize` axis #3 — see that ticket's
  `#5-regression`.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
