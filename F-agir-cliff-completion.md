---
id: F-agir-cliff-completion
title: "AGIR: complete the round-trip cliff — all currently unsupported anim node families"
status: DONE
severity: Medium
category: feature
tags: [agir, animgraph, roundtrip, subgraph, cliff]
---

# AGIR: complete the round-trip cliff

The Phase 3 AGIR compiler ships with a tiered subgraph cliff: state machines + anim layer interface override graphs round-trip, but several anim node families currently return `AGIR_SUBGRAPH_NOT_SUPPORTED` on compile (or are otherwise deliberately scoped out). Decompile already emits all of these correctly, so the cliff is asymmetric — round-trip is the missing half.

This ticket tracks completing each unsupported family. Implement these in any order; each is independent. Drop the corresponding `AGIR_SUBGRAPH_NOT_SUPPORTED` branch from `AGIRCompiler.cpp:EmitInstruction` as each lands.

## What's currently unsupported

### Compile-side cliffs (return `AGIR_SUBGRAPH_NOT_SUPPORTED`)

1. **`blend_space` blocks** — `UAnimGraphNode_BlendSpaceGraphBase` and its owned `UBlendSpaceGraph` + per-sample `UAnimationBlendSpaceSampleGraph` children. Decompile emits the full structure; compile rejects.
   - **Fix sketch:** add `CompileBlendSpaceInstruction` in `AGIRCompiler.cpp` that calls `FBlueprintEditorUtils::CreateNewGraph(OwnerNode, NAME_None, UBlendSpaceGraph::StaticClass(), <SampleSchema>::StaticClass())` for the owned graph, then recursively compiles `Children` against each sample sub-graph. Mirror the state-machine recursion in `CompileStateMachineChildren`.

2. **`layered_blend` blocks** — `UAnimGraphNode_LayeredBoneBlend` with its dynamic `BlendPoses` array of `FPoseLink`. Compile must grow the runtime array (`Node->Node.AddPose()` style) and call `Node->ReconstructNode()` so editor pins refresh, then wire pose links per index.
   - **Fix sketch:** new `CompileLayeredBlendInstruction`. Order matters: set runtime array length → reconstruct → wire pose links. The pin-name encoding is `BlendPose_<Index>` per the `FPoseLinkBase` array convention.

3. **Custom-transition graphs** (`LogicType == TLT_Custom`) — `UAnimStateTransitionNode::CustomTransitionGraph` of class `UAnimationCustomTransitionGraph`. Decompile emits but compile currently doesn't construct it.
   - **Fix sketch:** when `CompileTransitionInstruction` sees `logic_type=2` (TLT_Custom), invoke `Transition->CreateCustomTransitionGraph()` (engine method on `UAnimStateTransitionNode`). Recurse into the AGIR `Children` to populate the custom graph.

4. **`linked_anim` blocks** — `UAnimGraphNode_LinkedAnimGraph` and `UAnimGraphNode_LinkedAnimLayer`. References an external `UAnimBlueprintGeneratedClass` via `Node.InstanceClass + Node.LayerName`. No owned sub-graph, so compile is just node-creation + reflective field write of those two fields.
   - **Fix sketch:** add a `CompileLinkedAnimInstruction` that creates the editor node via `CreateAnimNode`, sets `InstanceClass` and `LayerName` via reflection (`AnimGraphConstructionUtils::WriteAnimNodeFieldByName`), then calls `Node->ReconstructNode()` so the function-signature-derived input pins refresh.

5. **`linked_input_pose` blocks** — `UAnimGraphNode_LinkedInputPose` (function-side counterpart of LinkedAnim). Used inside anim layer interface implementation graphs.
   - **Fix sketch:** node-creation + reflective writes for `Node.Name`, `InputPoseIndex`, `Inputs[]`, `FunctionReference`. The schema's `CreateFunctionGraphTerminators` already produces these on layer graph creation, so most often the AGIR text is reconstructing a graph the schema already populated — needs an "upsert" pattern (find existing by `Node.Name` first, write fields if found).

6. **`save_cached_pose` / `use_cached_pose` blocks** — `UAnimGraphNode_SaveCachedPose` and `UAnimGraphNode_UseCachedPose`. Linkage is by `CacheName` FString (per the auto-resolved decision in the plan).
   - **Fix sketch:** two-pass: Pass 1 creates Save nodes with `CacheName` set; Pass 2 creates Use nodes and resolves `Node.SaveCachedPoseNode` weak pointer via lookup against the Save name table. Both are simple node creation; the linkage is the only subtle part.

### Decompile-side gaps (decompiler classifies imprecisely)

7. **`AnimFunction` classification heuristic** — `FAGIRDecompiler::ClassifyGraph` always returns `AnimGraph` for non-layer top-level anim graphs. UE 5.6 stores anim function graphs and the main AnimGraph as plain `UAnimationGraph` instances on `Blueprint->FunctionGraphs`, indistinguishable at the graph level.
   - **Fix sketch:** distinguish by graph name and signature. The main AnimGraph is named `UEdGraphSchema_K2::GN_AnimGraph` (`"AnimGraph"`). Anim function graphs have arbitrary names AND contain a `UAnimGraphNode_LinkedInputPose` (or similar terminator pattern). Classify accordingly.

### Out-of-AGIR scope (separate IRs / cross-tool work)

8. **ControlRig RigVM IR** — `UAnimGraphNode_ControlRig` references a separate `UControlRigBlueprint`. AGIR currently records the rig class reference reflectively but doesn't enter the rig graph itself.
   - **Status:** out of AGIR's scope by design. File a separate `F-rgir-controlrig` ticket if/when authoring RigVM graphs becomes a goal.

9. **Anim notify track inline content** — referenced sequences carry `Notifies[]` and `AnimNotifyTracks[]`. AGIR only records the sequence asset path; notify content lives in the sequence asset's own dump.
   - **Status:** out of AGIR's scope — `UAnimSequenceBase` has its own dump path that should cover this. File a separate `F-asset-dump-anim-notifies` ticket if the sequence dump is incomplete.

10. **Inline transition-rule body** — currently AGIR records `rule="<graph_name>"` and the body is reachable via existing `blueprint.decompile`. The plan deliberately deferred inlining to avoid BPIR ↔ AGIR cross-IR coupling.
    - **Status:** out of AGIR's scope. Reasonable to revisit only if the round-trip workflow proves too clunky in practice.

11. **AnimBP auto-creation in compile** — `FAGIRCompiler::Compile` rejects missing target with `AGIR_TARGET_NOT_FOUND` because creating a `UAnimBlueprint` requires a `USkeleton` reference that AGIR text doesn't carry.
    - **Status:** out of scope — would require carrying skeleton metadata in the IR text, which conflicts with AGIR's "describe a graph, not metadata" shape. Separate `F-agir-with-target-spec` if desired.

## How to scope follow-up work

Each of (1)–(7) is roughly a 4–8h chunk for a developer familiar with the AGIR codebase. Items (1)–(6) are independent compile-side additions; (7) is a small decompiler change. Items (8)–(11) belong on separate tickets if revisited.

The current `FAGIRRoundTripStateMachineTest` (`TestAnimGraphHandlers.cpp`) silently passes on `AGIR_SUBGRAPH_NOT_SUPPORTED` against the Lyra mannequin AnimBP because the fixture trips multiple cliffs. **Each item above should land with a regression test against a synthetic fixture that exercises only that node family**, so the test becomes load-bearing as the cliffs are removed.

**Fix:** Implement the seven compile-side handlers and the AnimFunction classification heuristic in `AGIRCompiler.cpp` and `AGIRDecompiler.cpp` respectively. Ship each with a focused regression test under a synthetic fixture. Update the `Phase 3 ships tiered subgraph compile fidelity` decision in any future plan to reflect the new coverage.

## History
- `#1-initial-spec` `OPEN` reporter — Filed during AGIR Phase 3 sprint completion summary. Tracks all `AGIR_SUBGRAPH_NOT_SUPPORTED` cliffs (blend_space, layered_blend, custom-transition, linked_anim, linked_input_pose, save/use_cached_pose), the AnimFunction classification heuristic gap, and the out-of-AGIR-scope items (ControlRig RigVM, anim notifies, inline rule bodies, AnimBP auto-create) for cross-reference. Each compile-side item is independent; recommend implementing alongside a synthetic-fixture regression test rather than relying on the Lyra mannequin AnimBP fixture (which trips multiple cliffs and silently passes today).
- `#2-federated-snowglobe-implementation` `IN-REVIEW` developer — Implemented all seven items via the federated-snowglobe wave plan (`~/.claude/plans/f-agir-cliff-completion-federated-snowglobe.md`): Wave 1 promoted shared types into new `AGIRCliffHandlers.h`, split the `AGIR_SUBGRAPH_NOT_SUPPORTED` switch arm into per-opcode dispatches, and extended `FAGIRDecompiler::ClassifyGraph` with the AnimFunction heuristic (graph name != `GN_AnimGraph` AND contains `UAnimGraphNode_LinkedInputPose`). Wave 2 added six federated compile handlers in `AGIRCompiler_BlendSpace.cpp`, `_LayeredBlend.cpp`, `_LinkedAnim.cpp`, `_LinkedInputPose.cpp`, `_CachedPose.cpp` and inlined the TLT_Custom branch + custom-transition graph creation in `CompileTransitionInstruction`. Wave 3 extended decompile-side body emission in `AGIRTextEmitter.cpp` (block-opener form for `blend_space` with nested `sample_graph` blocks, conditional block form for `transition` with `custom_transition_body` blocks), introduced the `BlendSpaceSampleGraph` and `CustomTransitionBody` opcodes + parser block-opener support, added compile-side `Children` recursion (`CompileSampleGraph`, `CompileCustomTransitionBody`), and removed the load-bearing `AGIR_SUBGRAPH_NOT_SUPPORTED` tolerance plus the obsolete `FAGIRCompileRejectsBlendSpaceSubgraphTest`. Each family ships with a synthetic-fixture regression test under `Source/EditorAutomationRpcGatewayTests/Private/Assets/TestAGIR<Family>.cpp` (BlendSpace, LayeredBlend, LinkedAnim, LinkedInputPose, CachedPose, CustomTransition round-trips + AnimFunction classifier). Review pipeline (10 chunks × 3 reviewers + simplify pass, batched 4×) caught and fixed: missing `ApplyNodeGuidIfPresent` on blend_space + sample-graph children, missing `"guid"` in IgnoredKeys, missing `Inputs.Num()` round-trip assertion on LinkedInputPose, quote-stripping in CachedPose's `WriteUObjectFieldByName`, missing `AssignLocalIds` pre-pass in sample/body sub-emit functions, redundant `IdFor` pre-allocation, and silent `AddInfo` skips on missing Lyra fixture (now `AddWarning`). User runs `Automation RunTests EditorAutomationRpcGateway` to validate — `FAGIRRoundTripStateMachineTest` is now load-bearing on the Lyra mannequin AnimBP for the first time.
- `#3-verify-cliff-removed` `DONE` tester — Verified: `anim.decompile_agir` on `/Game/Characters/Heroes/Mannequin/Animations/ABP_Mannequin_Base` emits all previously-cliffed forms cleanly (`save_cached_pose`, `use_cached_pose`, `linked_anim`, `linked_input_pose`, `layered_blend`, full `state_machine` block with `transition ... logic_type=1` custom-transition variants) with zero `AGIR_SUBGRAPH_NOT_SUPPORTED` errors and empty `warnings`. Round-tripped a `save_cached_pose`+`AnimGraphNode_IdentityPose` snippet back via `anim.compile_agir` against the same AnimBP (mode=extend, save=false) → response `blocksCompiled=1, nodesCreated=2, warnings=[]`, confirming the compile-side handler accepts the new opcode without the cliff rejection. Both halves of the round-trip cliff are now load-bearing on the Lyra mannequin fixture.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. 1 citation sits in history rows and is left verbatim per the append-only rule. The one citation is in a history row and stays verbatim. It is a **pattern, not a path** — `Source/EditorAutomationRpcGatewayTests/Private/Assets/TestAGIR<Family>.cpp` — so it has no single successor; the directory it names is now `Source/PinWright/Private/Tests/Assets/`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
