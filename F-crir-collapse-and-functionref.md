---
id: F-crir-collapse-and-functionref
title: "CRIR — collapse sub-graphs, function library, and interface nodes"
status: DONE
severity: Low
category: feature
tags: [control-rig, rigvm, ir, crir, opcodes, collapse, function-library]
---

# CRIR — collapse sub-graphs, function library, and interface nodes

Four RigVM node kinds round-trip together because they share a single
parser-shape decision (recursive sub-graph block). Splitting them across
tickets would ship broken intermediate states.

- `URigVMCollapseNode` (`Nodes/RigVMCollapseNode.h`) — sub-graph wrapper
  with a `ContainedGraph`. Built via `URigVMController::CollapseNodes`
  (no flat `Add*`).
- `URigVMFunctionReferenceNode` (`Nodes/RigVMFunctionReferenceNode.h`) —
  reference into the asset's function library. Built via
  `AddFunctionReferenceNode` /
  `AddFunctionReferenceNodeFromDescription` /
  `AddExternalFunctionReferenceNode`.
- `URigVMFunctionEntryNode` / `URigVMFunctionReturnNode`
  (`Nodes/RigVMFunctionInterfaceNode.h` derivatives) — input/output stubs
  auto-created inside every collapse / function-library sub-graph. They
  cannot be added directly; CRIR must reconcile them when emitting the
  sub-graph body, not add them on compile.

## Resolved design decisions

- **Nested block shape — committed.** Introduce
  `rig_subgraph "<NodeName>" { ... }` as a recursive block kind inside a
  parent `rig_graph`. This matches the engine data model (one
  `URigVMGraph` per collapse / function) and lifts CRIR's brace depth
  past 1 for the first time. Hoisting siblings with parent-pointer
  attributes is rejected: it (a) forces every parser pass to two-phase
  resolve parent IDs, (b) diverges from BPIR composite-node convention,
  (c) AGIR already rejects sub-graphs with `AGIR_SUBGRAPH_NOT_SUPPORTED`
  — no in-codebase precedent for hoisting.
- **Entry/return reconciliation — committed.** Decompiler emits
  `function_entry` / `function_return` instructions as fixed first/last
  rows of every sub-graph body so the round-trip is textually symmetric.
  Compiler reconciles by *finding* the auto-created entry/return on the
  post-collapse graph and replaying pin defaults onto them, never calling
  an `Add*` (which would fail or duplicate).
- **Function library scaffolding — committed.** Add a new top-level
  `rig_function "<Name>" { ... }` block peer to `rig_graph` /
  `rig_hierarchy` (extend `ECRIREntryKind` with `RigFunction`). Resolved
  through `URigVMFunctionLibrary`'s controller via
  `GetControllerForGraph` + `AddFunctionToLibrary`. The `function_ref`
  opcode then names a function from that library.

**Fix:** Add `rig_subgraph` (nested) and `rig_function` (top-level) block
kinds, plus `collapse`, `function_ref`, `function_entry`,
`function_return` opcodes. Decompiler walks
`Collapse->GetContainedGraph()` recursively. Compiler routes through
`URigVMController::GetControllerForGraph` for the inner graph. Parser
extends its single-frame block tracker to an N-deep stack (port the
brace-stack pattern from `AGIRParser.cpp:666-714`).

## Depends on

- `F-crir-template-and-dispatch` — provides the dispatch-factory
  discrimination pattern reused for function-reference factory
  selection.

## History
- `#1-initial-spec` `OPEN` developer — Carved out from F-crir-extended-node-kinds wave 1. Both kinds need parser-shape design decisions (collapse: nested vs flattened rig_graph) and function-library scaffolding (function-reference: rig_function block kind) that don't belong in a single sprint subagent's scope.
- `#2-scope-expansion-and-decisions` `OPEN` developer — Audit added URigVMFunctionEntryNode / URigVMFunctionReturnNode to scope; they cannot ship separately from collapse/function bodies. Committed to nested `rig_subgraph` blocks (rejected hoisting) and a new top-level `rig_function` block. Entry/return reconciled by find-and-replay-defaults, not Add*. Ticket is now sprint-ready.
- `#3-collapse-functionref-implementation` `IN-REVIEW` developer — Implemented rig_subgraph/rig_function block kinds and collapse/function_ref/function_entry/function_return opcodes. Replaced scalar parser tracker with N-deep brace stack (FParseFrame array) ported from AGIRParser:666-714 — flexible enough that ticket 4 (control mutation) will reuse it for element trailing-{} sub-blocks. Extracted EmitGraphBody helper for recursive sub-graph emit; decompiler now emits function library via GetLocalFunctionLibrary. Entry/return reconciled by find-and-replay-pin-defaults on auto-created nodes (no Add*). function_ref compile splits on `::` for external vs same-asset. Tests: 4 round-trip cases across 3 files (single-level collapse, nested-two-level collapse, function ref library round-trip, entry/return pin default reconciliation).
- `#4-verify-fix` `DONE` tester — Verified: ran `controlrig.decompile_crir` on 3 real Mannequin assets (CR_Mannequin_Body, CR_Mannequin_BasicFootIK, CR_Mannequin_FootPlant). All emit the new `rig_function "<Name>" {…}` top-level blocks, `function_ref <Name>(…)` opcodes, and instruction-form `%nN = collapse "<Name>" @(x,y) {…}` sub-graphs with no `CRIR_UNSUPPORTED_NODE` warnings. (Note: a separate forward-reference compile bug surfaces when round-tripping these complex assets — `CRIR_UNDEFINED_REF` on `%nN` ids that are defined later in the same `rig_graph`. That's a wire-resolution-order bug independent of this ticket's scope; the new opcode emission and parsing themselves work as designed.)
