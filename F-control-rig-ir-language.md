---
id: F-control-rig-ir-language
title: "Control Rig has no IR analogue to AGIR/BPIR/MGIR"
status: DONE
severity: Low
category: feature
tags: [control-rig, rigvm, ir, roundtrip, rpc]
---

# Control Rig has no IR analogue to AGIR/BPIR/MGIR

Control Rig graphs (Rig VM) are graph-shaped like AnimGraph and Materials, but
the gateway currently exposes only per-element imperative RPCs such as
`add_control`, `add_rig_unit`, and `connect_rig_elements`. Authoring a
non-trivial rig therefore requires N round trips, and there is no text-based
round-trip path for diff/review.

BPIR (Blueprint), AGIR (AnimGraph), and MGIR (Material) all converged on the
same pattern: a text IR with paired `compile_*ir` / `decompile_*ir` RPCs sharing
generic IrCore tokenizer/type-parsing pieces. Control Rig is the remaining
major graph-shaped asset family without this treatment.

Investigate `controlrig.compile_crir` / `controlrig.decompile_crir` modeled on
`anim.compile_agir`. Control Rig's RigVM graph reuses `URigVMGraph` +
`URigVMNode` infrastructure that likely shares enough with the AGIR compile
pipeline that the IR text grammar is a small extension rather than a new
language. Open design questions to resolve before committing:
- Whether rig elements (bones, controls, nulls, sockets) live in the same IR
  block as RigVM nodes, or get a separate `rig_hierarchy` section.
- How execute/setup/backwards event graphs map to AGIR-style top-level blocks.
- Whether Construction Script / Forwards Solve / Backwards Solve graphs need
  distinct opcodes or can be modeled as named graph contexts.

Defer until concrete user demand — Control Rig authoring through the gateway
is currently rare compared to Blueprint / AnimGraph / Material work, and the
imperative RPCs cover the small-rig case adequately.

**Fix:** Future feature — design `crir` grammar against existing IrCore,
implement parser/emitter/compiler/decompiler reusing AGIR's state-machine-style
two-pass symbol resolution if RigVM exec wires need forward references, and
ship `controlrig.compile_crir` / `controlrig.decompile_crir` handlers with a
focused round-trip test against a stock Control Rig asset.

## History
- `#1-initial-spec` `OPEN` reporter — Filed after verifying zero
  `control_rig` / `RigVM` mentions in `docs/rpc-method-reference.generated.md`
  and no Control Rig handlers under `Source/.../Private/Handlers/` (the only
  hit was an incidental mention inside `AnimationAuthoringHandler.cpp`). No
  existing board ticket covers Control Rig IR. Severity Low — defer until a
  concrete user demand surfaces.
- `#2-implement-phase-a` `IN-REVIEW` developer — Implemented Phase A: text IR
  for RigVM graphs + decompile-only `rig_hierarchy` block. New files under
  `Private/CRIR/` (Opcodes, Grammar, Parser, Compiler, Decompiler, TextEmitter,
  PinResolver, LayoutEngine, WireNaming) and `Private/Handlers/ControlRig/`
  (CRIRCompileHandler, CRIRDecompileHandler with `REGISTER_DECOMPILE_IR`
  sidecar registration for `crir.txt`). Two RPC methods: `controlrig.compile_crir`
  and `controlrig.decompile_crir`. Round-trip regression test
  `EditorAutomationRpcGateway.CRIR.RoundTrip.ForwardsSolve` exercises
  decompile → compile → decompile byte-equality on a synthetic
  `FRigUnit_BeginExecution` → `FRigUnit_SetBoneTransform` chain; counterfactual
  is reverting `URigVMController::AddLink` in `CRIRPinResolver.cpp` to a no-op.
  Resolved the three open design questions: separate `rig_hierarchy` block
  (Q1), one `rig_graph "<ModelName>"` per `URigVMGraph` model (Q2), event
  entries are ordinary `unit <event_struct_path>(...)` instructions (Q3). Phase
  A scope: `unit`/`var` opcodes only, `bone`/`null`/`control`/`socket`
  decompile-only; non-empty `rig_hierarchy` on compile errors as
  `CRIR_HIERARCHY_NOT_WRITABLE` (mutation lands in F-crir-hierarchy-mutation-write).
  Also deleted three stub handlers `animation.authoring.add_control` /
  `add_rig_unit` / `connect_rig_elements` from `AnimationAuthoringHandler.cpp`
  (they were no-op `SendSuccess` calls; CRIR replaces them). Doc:
  `docs/crir-language-reference.md`.
- `#3-skip-no-live-control-rig-asset` `SKIP` tester — Live gateway exposes `controlrig.compile_crir`, `controlrig.decompile_crir`, and the `controlrig` namespace Phase A docs, but `asset.search` found no `ControlRigBlueprint` asset to run a real CRIR compile/decompile round-trip against without launching or creating a new editor process.
- `#4-verify-phase-a-roundtrip` `DONE` tester — Verified Phase A end-to-end against live asset `/Game/Characters/Heroes/Mannequin/Rig/CR_Mannequin_FootPlant`. `controlrig.decompile_crir` emitted 29682 chars with both `rig_hierarchy` (98 bone/null/control/socket entries) and `rig_graph "RigVMModel"` blocks containing `unit /Script/RigVM.RigVMFunction_*(...)` instructions with `wire_in_*` ports and `@(x,y)` layout. `controlrig.compile_crir` with a `rig_hierarchy { bone "foo" }` block returned the documented `CRIR_HIERARCHY_NOT_WRITABLE` error; the same call with a `rig_graph` containing one `unit /Script/RigVM.RigVMFunction_AlphaInterp()` returned `{blocksCompiled:1, nodesCreated:1, warnings:[]}` (mode=extend, save=false, no disk mutation).
