---
id: F-crir-deprecated-node-kinds-wontfix
title: "CRIR — deprecated If/Select/Branch/Array node kinds out of scope (wontfix)"
status: WONTFIX
severity: Low
category: feature
tags: [control-rig, rigvm, ir, crir, deprecated]
---

# CRIR — deprecated If/Select/Branch/Array node kinds out of scope (wontfix)

Four upstream `URigVMNode` leaf classes use the `UCLASS(Deprecated)` specifier
(C++ types are `UDEPRECATED_RigVMIfNode`, `UDEPRECATED_RigVMSelectNode`,
`UDEPRECATED_RigVMBranchNode`, `UDEPRECATED_RigVMArrayNode`; reflection strips
the prefix so they surface as `RigVMIfNode` / `RigVMSelectNode` /
`RigVMBranchNode` / `RigVMArrayNode` in the node-kind matrix).

The current `URigVMController` adders — `AddIfNode`, `AddSelectNode`,
`AddBranchNode`, `AddArrayNode` — no longer build instances of these classes.
They dispatch through `FRigVMDispatch_If`, `FRigVMDispatch_SelectInt32`, and
`FRigVMDispatch_Array*` templates, producing `URigVMDispatchNode` instances.
The deprecated leaf classes only appear on assets saved before the dispatch
migration that have not been load-time fixed up.

## Resolution

The real round-trips for these surfaces are already covered:
- `CRIR.RoundTrip.IfSelect` — `URigVMDispatchNode` instances built by
  `AddIfNode` / `AddSelectNode`.
- `CRIR.RoundTrip.TemplateAndDispatch` — `URigVMDispatchNode` `ArrayAdd` arm.

CRIR does not emit If/Select/Branch/Array opcodes against the deprecated
classes and does not provide a compiler path that creates them. The
decompiler's existing `# TODO unsupported node kind:` arm is the correct
round-trip output if a pre-dispatch-era asset still has any of these nodes
after load-time fixup.

Filed explicitly so future audits of the `URigVMNode` taxonomy don't keep
re-discovering these gaps and re-opening them. Mirrors the prior
`F-crir-parameter-node-wontfix` decision for the same `UCLASS(Deprecated)`
pattern.

## History
- `#1-resolved-wontfix` `WONTFIX` developer — Engine-deprecated leaf classes; controller adders now build URigVMDispatchNode via dispatch templates and the dispatch surface is covered by CRIR.RoundTrip.IfSelect and CRIR.RoundTrip.TemplateAndDispatch. Recording wontfix so future audits don't reopen.
