---
id: F-agir-state-alias
title: "AGIR: state_alias opcode for UAnimStateAliasNode"
status: DONE
severity: Medium
category: feature
tags: [agir, animgraph, roundtrip, state-machine, cliff]
---

# AGIR: state_alias opcode for UAnimStateAliasNode

The Phase 3 AGIR cliff completion (`F-agir-cliff-completion`) shipped six federated compile handlers + the AnimFunction classifier, but missed `UAnimStateAliasNode`. Lyra's `ABP_Mannequin` `LocomotionSM` uses alias nodes (`PivotSources`, `JumpSources`, `JumpFallInterruptSources`, `IdleAlias`, `CycleAlias`) as transition endpoints, so the `RoundTripStateMachine` test failed with `AGIR_SYMBOL_NOT_FOUND: Transition 'PivotSources -> Pivot' references unknown state` — the decompiler silently dropped the alias node, leaving the recompile referencing an undefined symbol.

`UAnimStateAliasNode` derives from `UAnimStateNodeBase` (sibling of `UAnimStateNode` and `UAnimStateConduitNode`). It carries `bGlobalAlias` (bool) and `AliasedStateNodes` (`TSet<TWeakObjectPtr<UAnimStateNodeBase>>`). When `bGlobalAlias` is true, the alias represents "any state" and `AliasedStateNodes` is empty.

**Fix:** Add a sibling opcode mirroring the `state` / `conduit` shape:
- `AGIROpcodes.h` — new `EAGIROpcode::StateAlias`, validated as state-machine-only.
- `AGIRGrammar.cpp` — register the `state_alias` keyword in the opcode map and the reverse text map.
- `AGIRParser.cpp` — add `StateAlias` to the name-leading opcode set so the alias name reads into `SymbolName`.
- `AGIRTextEmitter.cpp` — extend `EmitStateMachine`'s Pass A loop with an `EmitStateAlias` branch that produces a single-line block `state_alias <Name> aliases="State1,State2" global_alias=true` (alias targets sorted for deterministic output; comma-separated value quoted to survive the whitespace splitter).
- `AnimGraphConstructionUtils.h/.cpp` — `CreateStateAlias(MachineGraph, AliasName, Position)` mirroring `CreateConduit`, sets `StateAliasName` so `GetStateName()` returns the AGIR symbol name.
- `AGIRCompiler.cpp` — `CompileStateAliasInstruction` registers the alias name into `StateSymbols` (so transitions referencing the alias resolve like states/conduits) and writes `bGlobalAlias`. Add a Pass C `RebindStateAliasTargets` that walks compiled aliases after Pass B and binds `GetAliasedStates()` from the comma-split `aliases=` value against the now-complete `StateSymbols` map (aliases can reference states declared later in the SM, so the bind has to be deferred).

## History
- `#1-initial-spec` `OPEN` reporter — Filed during `mcp-test-loop` cycle 1 against Lyra `ABP_Mannequin` `LocomotionSM`. The state_alias gap was not in `F-agir-cliff-completion` because that ticket scoped to anim-node families; alias is a state-machine-internal construct that slipped through.
- `#2-implemented-end-to-end` `IN-REVIEW` developer — Implemented the opcode end-to-end across `AGIROpcodes.h`, `AGIRGrammar.cpp`, `AGIRParser.cpp`, `AGIRTextEmitter.cpp`, `AnimGraphConstructionUtils.h/.cpp`, and `AGIRCompiler.cpp`. The Pass A registration + Pass C rebind pattern matches the existing state/conduit/transition phasing. After this landed, the same Lyra fixture advanced past `AGIR_SYMBOL_NOT_FOUND` and surfaced the next cliff (`AGIR_TARGET_NOT_FOUND: no implemented interface declares function 'FullBodyAdditives'`) — tracked separately in `F-agir-interface-function-decls`.
- `#3-verify-fix` `DONE` tester — Verified: `anim.decompile_agir` on `/Game/Characters/Heroes/Mannequin/Animations/ABP_Mannequin_Base` emits all five expected aliases in `LocomotionSM` (`state_alias PivotSources aliases="Cycle,Start"`, `JumpSources aliases="Cycle,Idle,Pivot,Start,Stop"`, `IdleAlias aliases="Idle"`, `CycleAlias aliases="Cycle"`, `JumpFallInterruptSources aliases="FallLand,FallLoop,JumpApex,JumpStart,JumpStartLoop"`) with alphabetically sorted comma-separated targets. Transitions referencing aliases (`PivotSources -> Pivot`, `JumpSources -> JumpSelector`, `EndInAir -> IdleAlias`, `EndInAir -> CycleAlias`, `JumpFallInterruptSources -> EndInAir`) all resolved — no `AGIR_SYMBOL_NOT_FOUND`. `warnings: []`.
