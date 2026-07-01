---
id: F-crir-parameter-node-wontfix
title: "CRIR — parameter node explicitly out of scope (wontfix)"
status: WONTFIX
severity: Low
category: feature
tags: [control-rig, rigvm, ir, crir, deprecated]
---

# CRIR — parameter node explicitly out of scope (wontfix)

`URigVMParameterNode` (`Nodes/RigVMParameterNode.h`) carries
`UCLASS meta=(Deprecated="5.1")`. Both `AddParameterNode` and
`AddParameterNodeFromObjectPath` on `URigVMController` are tagged
`DeprecatedFunction`. The engine has replaced parameter nodes with
variable nodes since 5.1.

## Resolution

CRIR does not emit a parameter opcode and does not provide a compiler
path that creates one. The decompiler's existing
`# TODO unsupported node kind:` arm is the correct round-trip output if a
5.1-era asset still has parameter nodes after load-time fixup.

Filed explicitly so future audits of the `URigVMNode` taxonomy don't keep
re-discovering this gap and re-opening it.

## History
- `#1-resolved-wontfix` `WONTFIX` developer — Engine-deprecated since UE 5.1; controller adders tagged DeprecatedFunction. Decompiler's existing TODO arm covers any stray legacy data. Recording wontfix so future audits don't reopen.
