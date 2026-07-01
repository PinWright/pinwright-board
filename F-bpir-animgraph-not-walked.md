---
id: F-bpir-animgraph-not-walked
title: "BPIR decompiler does not walk AnimationGraphs / AnimationFunctions — entire AnimGraph BP family is invisible to BPIR"
status: DONE
severity: High
category: feature
tags: [bpir, decompiler, animgraph, coverage-gap]
---

# AnimGraph BPs are entirely outside BPIR scope

The BPIR decompiler walks only three page arrays on `UBlueprint`:
- `UbergraphPages`
- `FunctionGraphs`
- `MacroGraphs`

Any `UAnimBlueprint`-derived asset (player anim BPs, weapon anim BPs,
montage-driver BPs, anim layer interfaces) decompiles with **only its
event-graph and macro logic visible**. The actual animation graph —
state machines, blend trees, anim notifies, IK solvers, control rig
nodes — is silently dropped. BPIR text shows nothing about the asset's
primary purpose.

## Factual corrections vs the original spec

The original `#1-initial-spec` body cited two things that are wrong on
UE 5.6 and are corrected here:

1. **There are no `Anim*` UPROPERTYs on `UAnimBlueprint`.** The fields
   originally named (`AnimationGraphs`, `AnimationFunctions`,
   `AnimGraphLayers`) do not exist. Anim graphs live on the inherited
   `UBlueprint::FunctionGraphs` array and are identified by schema
   (`UAnimationGraphSchema`-derived). The page-list view is exposed at
   runtime via `UAnimationBlueprintLibrary::GetAnimationGraphs`. Anim
   layer interface override pose graphs live under
   `Blueprint->ImplementedInterfaces[i].Graphs`.
2. **The canonical bulk-walk helper is
   `FBlueprintEditorUtils::GetAllNodesOfClass<T>(BP, OutNodes)`**, not
   any per-array iteration. It already handles inherited graphs, sub-
   graphs, state machines, and ImplementedInterfaces graphs in one call.

## Three-phase plan

This ticket is implemented across three phases, one coordinated piece
of work. Full plan:
`C:\Users\Alexander\.claude\plans\plan-first-floofy-meadow.md`.

- **Phase 1 — `anim_graph.json` aspect.** Read-only flat picture of
  what an anim BP contains: pages (with kind classification), state
  machines (with states / transitions / conduits), and the multiset of
  `UAnimGraphNode_*` classes used. Surfaced as a new aspect on
  `asset.dump`. Unblocks LLM-driven inspection workflows immediately.
- **Phase 2 — AGIR decompile-only IR.** A parallel IR under
  `Private/AGIR/`, mirroring `Private/MGIR/`. Reuses `IrCore/`
  substrate. Covers anim node families via reflection over runtime
  `FAnimNode_*` structs plus narrow special opcodes for
  `state_machine`, `blend_space`, `layered_blend`, `linked_anim`,
  `save_cached_pose`, `use_cached_pose`. Decompile-only on this phase;
  surfaced as a separate `agir.txt` dump aspect plus a
  `anim.decompile_agir` RPC.
- **Phase 3 — AGIR round-trip compile.** Tiered subgraph fidelity:
  top-level anim graphs, anim layer interface override graphs, and
  state machines round-trip; blend-space / blend-space-sample / layered-
  blend / custom-transition subgraphs return
  `AGIR_SUBGRAPH_NOT_SUPPORTED` on compile (decompile-only — explicit
  cliff, mirrors MGIR's composite cliff). Surfaced as
  `anim.compile_agir`. BPIR is touched only by a single one-line
  schema-filter guard at the page-array loops; otherwise BPIR is
  unchanged.

ControlRig RigVM IR, blend-space sample subgraph round-trip, custom-
transition graph round-trip, and implicit `UAnimBlueprint` creation
(needs a `USkeleton` AGIR text can't carry) are explicitly out of scope.

## Repro

Any `UAnimBlueprint` asset (e.g. PDS character / weapon anim BPs).
Decompile via `blueprint.decompile`; observe BPIR contains only event-
graph logic, no state machines or blend trees. After Phase 1, the same
asset's `asset.dump` folder carries an `anim_graph.json` listing pages,
state machines, and node-class counts. After Phase 1's BPIR schema
filter lands, BPIR text for an anim BP is empty (or covers only K2
graphs like the event graph) instead of partial — no half-walked anim
content.

## History
- `#1-initial-spec` `OPEN` reporter — K2Node catalog audit confirmed BPIR walks only `UbergraphPages` / `FunctionGraphs` / `MacroGraphs`; AnimGraph pages and AnimGraph-specific node families (~80 `UAnimGraphNode_*` subclasses + state machines + transitions) are entirely outside scope. Major coverage hole for any project using anim BPs. Phased proposal: dump aspect first, then dedicated AGIR, then round-trip.
- `#2-reformulated-three-phase-plan` `OPEN` reporter — Corrected two factual errors in the original body: there are no `Anim*` UPROPERTYs on `UAnimBlueprint` (anim graphs live on `UBlueprint::FunctionGraphs` filtered by `UAnimationGraphSchema`; anim layer interface graphs under `Blueprint->ImplementedInterfaces[].Graphs`); canonical bulk walker is `FBlueprintEditorUtils::GetAllNodesOfClass<T>`, not per-array iteration. Restated scope as three coordinated phases (anim_graph.json aspect, AGIR decompile, AGIR round-trip with tiered subgraph cliff). Full plan at `C:\Users\Alexander\.claude\plans\plan-first-floofy-meadow.md`.
- `#3-implemented-phase-1` `IN-REVIEW` developer — Phase 1 landed. New files: `Private/Handlers/Animation/AnimGraphDumpBuilder.h`, `Private/Handlers/Animation/AnimGraphDumpBuilder.cpp`, `EditorAutomationRpcGatewayTests/Private/Utility/TestAssetDumpAnimGraph.cpp`. Edited: `Private/Handlers/Asset/AssetDumpHandler.h` (`DumpFileNames::AnimGraph` constant), `Private/Handlers/Asset/AssetDumpHandler.cpp` (new `UAnimBlueprint` branch before the generic `UBlueprint` branch; `anim_graph.json` added to canonical baseline), `Private/Decompiler/BpirDecompiler.cpp` and `Private/Compiler/BpirCompiler.cpp` (schema filter at the page-array loops to refuse anim graphs; symmetric on decompile and compile). New aspect `anim_graph.json` carries three sections: `pages[]` (per-graph `name`/`guid`/`kind`/`parent`, kind ∈ {AnimGraph, AnimLayer, StateMachineGraph, StateGraph, TransitionRule, CustomTransition, Conduit, BlendSpaceGraph, BlendSpaceSample}), `state_machines[]` (per-machine states + transitions with priority/rule_graph/bidirectional/disabled + conduits), `anim_node_classes[]` (sorted `{class, count}` multiset).
- `#4-verified-anim-graph-dump` `DONE` tester — Verified: `asset.dump` on `/Game/Characters/Heroes/Mannequin/Animations/ABP_Mannequin_Base` wrote `anim_graph.json` with non-empty `pages`, `state_machines`, and `anim_node_classes` (including `LocomotionSM` and `/Script/AnimGraph.AnimGraphNode_StateMachine`), while `bpir.txt` kept K2 function/event graph output and did not emit the AnimGraph state-machine body. Test: `mcp__editor_automation__.call path="asset.dump" args={"assetPath":"/Game/Characters/Heroes/Mannequin/Animations/ABP_Mannequin_Base"}` then inspected `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Characters/Heroes/Mannequin/Animations/ABP_Mannequin_Base/anim_graph.json` and `bpir.txt`.
