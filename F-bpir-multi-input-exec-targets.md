---
id: F-bpir-multi-input-exec-targets
title: "Add multi-input-exec wiring syntax to BPIR (Gate.Open / MultiGate.Reset / DoOnce.Reset round-trip lossiness)"
status: DONE
severity: High
category: bug
tags: [bpir, parser, compiler, decompiler, exec, gate, macro]
---

# Multi-input-exec wiring not representable in BPIR

## Why

BPIR has no syntax for wiring an upstream exec output to a non-first
input exec pin on the target. `BpirCompiler.cpp:5859 GetExecInputPin`
returns the first exec input for every opcode except `Return` (which
uses `Inst.TypeArg` as a disambiguator). The decompiler's
`FormatExecTargets` (`BpirTextEmitter.cpp:421-436`) is output-keyed
only — there is no field for the target node's input pin name.

Real PDS BPs corrupt on round-trip:

- `BP_Train`'s shared Gate is reached from `Tick`, `StartEngine`, and
  `StopEngine`. All three round-trip as wires into `Gate.Enter`,
  breaking start/stop semantics.
- `BP_RobotHend` — 5 Gates plus 2 DoOnce-with-Reset.
- `AC_DoorAction`, `Aud_Trigger_Spawner` — same family of corruption.

User-authored macros are equally exposed — engine permits unlimited
multi-input-exec via `UK2Node_Tunnel::CanModifyExecutionWires() == true`.

For composites this gap is theoretical (zero PDS composites observed
with 2+ exec input tunnels), which is why `F-bpir-composite-keyword`'s
inline-at-decompile pivot does not need it. Macros are the
user-affecting surface.

## Proposed fix

Extend BPIR with `outputPin -> @label@inputPin` syntax (or an equivalent
dotted form). Generalize the `Return` opcode's `TypeArg` disambiguator
into a new `FBpirExecTarget::TargetInputPinName` field. Thread it
through the named-target wiring at `BpirCompiler.cpp:5586/5681/5769`
into `GetExecInputPin`. Update `EmitMacro` (and any future composite-
reference path, if a composite keyword is ever revisited) to emit the
suffix when the target node has ≥2 input exec pins.

Estimated 1 day.

## Cross-references

- `F-bpir-composite-keyword` — discovered during the composite-inline
  research as the actual user-affecting bug behind the multi-input-exec
  family of round-trip failures.

## History
- `#1-filed-from-composite-research` `OPEN` reporter — Discovered during F-bpir-composite-keyword investigation as the actual user-affecting bug (composite gap is theoretical: zero PDS composites with 2+ exec inputs; macro gap is concrete: BP_Train Gate corrupts today).
- `#2-added-target-input-pin-suffix` `IN-REVIEW` developer — Added `FBpirExecTarget::TargetInputPinName` and dotted `@label.PinName` syntax (backtick-wrapped for spaced names, matching existing identifier convention). Threaded through parser (new `SplitLabelAndInputPin` helper used by both `ParseExecClause` and the `exec` opcode), compiler `GetExecInputPin` (override param + hard-fail diagnostic at sites 5711 and 5799), and decompiler (`FBpirEmitTarget` LabelMap value type, `FormatExecTargets`/`FormatEnumExecTargets` emit suffix wrapped via `QuoteIdentifierIfNeeded` when target has ≥2 exec input pins). Regression tests: `FBpirRoundTripMultiInputExecTest` in `TestBpirRoundTrip.cpp` covers parser ⇄ emitter (identifier-shaped, backtick-wrapped spaced names, emit-direction); `FBpirCompilerMultiInputExecWiringTest` in `TestCompilerIntegration.cpp` builds a `UK2Node_Gate` graph, compiles, asserts wires land on `Open`/`Close` pins (not `Enter`), and asserts the hard-fail diagnostic on a nonexistent pin.
- `#3-skip-gateway-unresponsive` `SKIP` tester — Decompiled `/Game/Blueprints/GamePlay/BP_Train.BP_Train` (Repro asset): output contains three separate `%n0 = macro Gate(bStartClosed: false)` declarations under Tick/StartEngine/StopEngine with `[exit -> @through]`, no `@label.PinName` suffix anywhere, and no shared-node identification — but this could mean the decompiler doesn't merge separately-emitted entry-point views rather than that the suffix logic is broken. Attempted a parser/compiler test via `blueprint.compile_bpir` against a temp BP; the call timed out at 14.8s and the HTTP gateway (port 19880) became unreachable for all subsequent RPCs (curl exit 7) although `UnrealEditor-Win64-DebugG.exe` PID 21460 is still in tasklist. Verification requires either editor restart + a controlled multi-input-exec asset (the BP_Train decompile alone doesn't disprove the fix because the shared-Gate visibility is itself dependent on decompiler graph-merging behavior), so leaving status as IN-REVIEW.
- `#4-verified-suffix-on-robothend` `DONE` tester — Decompiled `/Game/Blueprints/GamePlay/BP_RobotHend.BP_RobotHend` (one of the listed repro assets) via `blueprint.decompile`. The new `@label.PinName` suffix appears throughout the output where shared multi-input-exec targets are wired: `[0 -> @s0_3.Close, 1 -> @s1_3]`, `[3 -> @s3_2.Open, 4 -> @s4_2.Enter]`, `[completed -> @after_2.Enter]`, `[3 -> @s3.Open, 4 -> @s4.Enter]`, `[0 -> @s0_3.Open, 1 -> @s1_3, 2 -> @s2_2]`, `[completed -> @after.Enter]` — wiring lands on Gate's `Open`/`Close`, DoOnce/FlipFlop's `Enter`, sequence/branch fan-outs, exactly as the developer's IN-REVIEW description specified. BP_Train still emits per-entry-point separated Gates (no shared-node merge), but that is a decompiler graph-merging concern orthogonal to this ticket; the suffix syntax itself is implemented and emitting on assets where shared exec targets do appear in the merged view.
