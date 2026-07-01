---
id: F-crir-extended-node-kinds
title: "CRIR — extended RigVM node kinds (reroute, comment, if, select, template, enum, collapse, function-ref, invoke-entry)"
status: DONE
severity: Low
category: feature
tags: [control-rig, rigvm, ir, crir, opcodes]
---

# CRIR — extended RigVM node kinds

CRIR Phase A (shipped via `F-control-rig-ir-language`) recognizes only two
RigVM node kinds: `unit` (`URigVMUnitNode`, including event entries) and
`var` (`URigVMVariableNode`). The decompiler emits a
`# TODO unsupported node kind: <ClassName>` comment and a warning for every
other node kind it encounters; the compiler errors with
`CRIR_UNSUPPORTED_NODE_KIND` if any of them appear in input text.

Phase A is sufficient for hand-written functional rigs (event entry → rig
units). Real-world Control Rig assets in Lyra and PDS use the extended kinds
extensively for graph organization (reroutes, comments) and structured
control flow (if/select). Round-tripping those assets requires Phase B
opcode support.

The full out-of-scope set, in rough priority order:

- `URigVMRerouteNode` — graph-routing pass-through. Common; cheap to
  implement (single input/output pin, type carried on the pin). Likely first.
- `URigVMCommentNode` — author commentary. Position + size + text only;
  trivial.
- `URigVMIfNode`, `URigVMSelectNode` — structured control flow. Both have
  fixed pin shapes; serialize as one opcode each.
- `URigVMTemplateNode` — pre-resolution template instances. Closest to `unit`
  but with type parameters that need a separate text grammar slot.
- `URigVMEnumNode` — enum-typed value source. Small.
- `URigVMCollapseNode` — sub-graph wrapper. Contains a `URigVMGraph*` of its
  own; this is the only kind in this list that recursively contains another
  graph and therefore requires either nested `rig_graph` blocks or a separate
  flattening strategy. **Has a real design question** — see below.
- `URigVMFunctionReferenceNode` — references a function in the asset's
  function library. Phase A's compiler already restricts model walk to
  top-level non-library models, so the function library itself is also out of
  scope until Phase B.
- `URigVMInvokeEntryNode` — invokes a named entry from another graph in the
  same asset. Small.

## Open design questions

- **Collapse-node sub-graphs:** nested `rig_graph "<NodeName>" { ... }` block
  inside a parent `rig_graph` block, or hoist all sub-graphs to siblings at
  the top level with a parent-pointer attribute? Nesting matches the engine
  data model directly but complicates the parser (brace stack depth >1 for
  the first time in CRIR); flattening is parser-cheap but loses visual scope.
  AGIR and BPIR can be consulted for precedent — both deal with nested
  function/state graphs and chose different answers.
- **Function-library opcode:** likely a new top-level `rig_function "<Name>" { ... }`
  block kind, following the `rig_graph` shape but resolved against the
  function library's controller instead of the model controller.

**Fix:** Wave 1 shipped the six lowest-complexity kinds: `reroute`,
`comment`, `if`, `select`, `enum`, `invoke_entry`. Each is a new
`ECRIROpcode` entry with parser/emitter/compiler/decompiler branches
calling the existing `URigVMController` API (`AddFreeRerouteNode`,
`AddCommentNode`, `AddIfNode`, `AddSelectNode`, `AddEnumNode`,
`AddInvokeEntryNode`). Five new round-trip tests under
`Source/.../Private/Tests/Assets/TestCRIR{Reroute,Comment,IfSelect,Enum,InvokeEntry}.cpp`
exercise each kind group. Carved out:

- `template` (bare `URigVMTemplateNode` plus the general `URigVMDispatchNode`
  beyond if/select) → `F-crir-template-and-dispatch`. The blocker is text
  grammar disambiguation from the existing `unit` opcode, since
  `URigVMUnitNode : URigVMTemplateNode`.
- `collapse` (`URigVMCollapseNode`) + `function_ref`
  (`URigVMFunctionReferenceNode`) → `F-crir-collapse-and-functionref`.
  Both need parser-shape design decisions (nested vs flattened sub-graph
  blocks; function-library opcode introduction) the original ticket flagged
  as out-of-scope for one sprint.

## Depends on

- `F-control-rig-ir-language` (Phase A) must have landed.

## History
- `#1-initial-spec` `OPEN` developer — Filed as the explicit Phase B carve-out
  during the Phase A sprint per the no-silent-defer rule. Phase A capped at
  `unit` + `var` opcodes to keep the first ship under one sprint; the
  remaining opcodes are needed for round-tripping real-world rigs but each is
  small enough to land independently. Defer until a concrete user request
  identifies which kind is blocking.
- `#2-wave-1-six-kinds-shipped` `IN-REVIEW` developer — Wave 1: shipped reroute/comment/if/select/enum/invoke_entry as new ECRIROpcode entries with parser/emitter/compiler/decompiler branches. Deferred template (overlaps URigVMUnitNode disambiguation), collapse, function-ref to follow-up tickets F-crir-template-and-dispatch and F-crir-collapse-and-functionref. Five new round-trip tests cover each kind group.
- `#3-verify-fix` `DONE` tester — Verified: CRIROpcodes.h enum contains all 6 new entries (Reroute/Comment/If/Select/Enum/InvokeEntry); five test files present (TestCRIRReroute/Comment/IfSelect/Enum/InvokeEntry.cpp); CRIRCompiler.cpp calls AddFreeRerouteNode/AddCommentNode/AddIfNode/AddSelectNode/AddEnumNode/AddInvokeEntryNode. Live `controlrig.decompile_crir` on `/Game/Characters/Heroes/Mannequin/Rig/CR_Mannequin_Procedural` emits 6 reroute + 4 comment instructions with zero `TODO unsupported node kind` markers (remaining TODOs are `unsupported element kind: Curve`, which is out-of-scope hierarchy work).
