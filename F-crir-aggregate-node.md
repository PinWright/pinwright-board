---
id: F-crir-aggregate-node
title: "CRIR — aggregate node coverage"
status: DONE
severity: Low
category: feature
tags: [control-rig, rigvm, ir, crir, opcodes, aggregate]
---

# CRIR — aggregate node coverage

`URigVMAggregateNode` (`Nodes/RigVMAggregateNode.h`) is a specialization
of `URigVMCollapseNode` for N-ary commutative operators (IntAdd, FloatMul, …).
It's gated engine-side by `UE_RIGVM_AGGREGATE_NODES_ENABLED` and is built
implicitly when adding aggregate pins to a unit node, not via a top-level
`URigVMController::Add*` helper.

The decompiler currently emits aggregate nodes as
`# TODO unsupported node kind: RigVMAggregateNode`.

## Resolution

Two acceptable approaches; pick at implementation time:

1. **Free-ride on collapse.** Once `F-crir-collapse-and-functionref` lands,
   aggregate nodes round-trip as plain collapse sub-graphs — the interior
   unit nodes preserve runtime behavior. The only thing lost is the "this
   collapse is actually an aggregate" engine-side flag, which is metadata
   the user cannot author directly anyway.
2. **Dedicated `aggregate <InnerStructPath>` opcode.** Single-line
   shorthand that on compile calls `AddAggregatePin` repeatedly. Lossy
   when authoring by hand because the pin count comes from the unit's
   aggregate metadata, not from CRIR text. Adds grammar surface for a
   flag users don't directly touch.

**Recommended:** ship option 1 (free with the collapse ticket). File a
follow-up only if aggregate-flag round-trip turns out to matter for a real
Control Rig asset in the project.

**Fix:** No new opcode unless option 2 wins. Update the decompiler's
`# TODO unsupported` arm to recognize `URigVMAggregateNode` and route it
through the same code path as `URigVMCollapseNode`.

## Depends on

- `F-crir-collapse-and-functionref` (must ship first).

## History
- `#1-initial-spec` `OPEN` developer — Spun out during full-coverage audit of URigVMNode taxonomy. Aggregate is engine-gated by UE_RIGVM_AGGREGATE_NODES_ENABLED and is a CollapseNode subclass, so it naturally rides on the collapse ticket. Filed separately so the audit's taxonomy is closed and the decision is recorded.
- `#2-shape-a-comment-and-test` `IN-REVIEW` developer — Shipped Shape A (zero new code) per analysis. URigVMAggregateNode rides the ticket-2 collapse arm via Cast<URigVMCollapseNode> inheritance — no explicit aggregate Cast needed. Added a documentation comment on the collapse arm in CRIRDecompiler.cpp pointing future readers at the round-trip test, plus a sentinel test appended to TestCRIRCollapseRoundTrip.cpp that asserts an authored aggregate decompiles to rig_subgraph (not TODO) and recompiles to a working collapse — accepting the documented class-identity loss (aggregate → plain collapse). Test guarded with #if UE_RIGVM_AGGREGATE_NODES_ENABLED.
- `#3-verify-fix` `DONE` tester — Verified: confirmed CRIRDecompiler.cpp:725-731 has the documented collapse-arm comment naming URigVMAggregateNode and pointing at the test; confirmed TestCRIRCollapseRoundTrip.cpp:231-380 contains FCRIRDecompiler_AggregateNode_RoundTripsAsCollapse_DegradedClass with the no-TODO and rig_subgraph-present assertions, guarded by UE_RIGVM_AGGREGATE_NODES_ENABLED. Ran system.run_tests with test="EditorAutomationRpcGateway.CRIR.RoundTrip.AggregateAsCollapse" — job completed with has_errors=false, resolvedTests matched requestedTests, no missingTests.
