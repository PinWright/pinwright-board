---
id: F-crir-coverage-test-matrix
title: "CRIR — node-kind coverage test matrix"
status: DONE
severity: Low
category: feature
tags: [control-rig, rigvm, ir, crir, tests]
---

# CRIR — node-kind coverage test matrix

After `F-crir-template-and-dispatch`, `F-crir-collapse-and-functionref`,
and `F-crir-aggregate-node` land, every concrete `URigVMNode` leaf must
have a round-trip test asserting decompile → recompile produces an
equivalent graph. Visual layout (positions, comment colors) is exempt per
the project's IR-equivalence policy.

## Coverage required (one fixture rig per row)

| Class | Opcode |
|---|---|
| `URigVMUnitNode` | `unit` |
| `URigVMTemplateNode` (bare) | `template` |
| `URigVMDispatchNode` (if) | `if` |
| `URigVMDispatchNode` (select) | `select` |
| `URigVMDispatchNode` (branch) | `dispatch RigVMDispatch_Branch` |
| `URigVMDispatchNode` (array add) | `dispatch RigVMDispatch_ArrayAdd` |
| `URigVMVariableNode` | `var` |
| `URigVMCommentNode` | `comment` |
| `URigVMRerouteNode` | `reroute` |
| `URigVMEnumNode` | `enum` |
| `URigVMInvokeEntryNode` | `invoke_entry` |
| `URigVMCollapseNode` (single level) | `collapse` + `rig_subgraph` |
| `URigVMCollapseNode` (two-level nested) | `collapse` + nested `rig_subgraph` |
| `URigVMFunctionReferenceNode` | `function_ref` + `rig_function` |
| `URigVMAggregateNode` | (rides on collapse) |

Plus, for `F-crir-control-mutation-write`, one fixture per `FRigControlValue`
variant (Bool/Float/Int/Vector2D/Vector/Rotator/Transform/EulerTransform)
and one maximal-settings control fixture exercising every non-default
sub-block field.

## Fix

Add `IMPLEMENT_SIMPLE_AUTOMATION_TEST` cases under
`Source/EditorAutomationRpcGateway/Private/Tests/CRIR/` — one per row above.
Each test loads or constructs the fixture rig, runs `FCRIRDecompiler` →
`FCRIRCompiler`, asserts node-class + pin-default + link parity. Provides
a coverage gate so future taxonomy drift fails loudly.

## Depends on

- `F-crir-template-and-dispatch`
- `F-crir-collapse-and-functionref`
- `F-crir-aggregate-node`
- `F-crir-control-mutation-write`

## History
- `#1-initial-spec` `OPEN` developer — Audit gap: nothing currently asserts the full URigVMNode taxonomy is closed. With four wave-2 tickets adding coverage for ~10 new node kinds, an explicit matrix prevents regressions when the engine adds a new leaf class.
- `#2-matrix-completed-and-gate-added` `IN-REVIEW` developer — Audit found all matrix rows except `URigVMVariableNode` already had round-trip tests (TestCRIRRoundTrip / TestCRIRTemplateAndDispatch / TestCRIRIfSelect / TestCRIRComment / TestCRIRReroute / TestCRIREnum / TestCRIRInvokeEntry / TestCRIRCollapseRoundTrip / TestCRIRFunctionRefRoundTrip; control values via TestCRIRControlMutation). Added the missing `Tests/Assets/TestCRIRVariableNode.cpp` (`CRIR.RoundTrip.Variable`) which pre-registers a `MyBool` member variable on both source and target BPs (required because `URigVMController::AddVariableNode` errors out when the variable is unregistered) and asserts byte-equal CRIR text round-trip. Added `Tests/Assets/TestCRIRNodeKindCoverage.cpp` (`CRIR.Coverage.NodeKindMatrix`) — a reflection-based gate that enumerates concrete `URigVMNode` UClass subclasses via `TObjectIterator<UClass>` and fails loudly if any class lands outside the Covered/Uncovered/StructuralBases sets, satisfying the "taxonomy drift" intent. Tests live under `Tests/Assets/` (existing convention) rather than `Tests/CRIR/` named in the ticket body.
- `#3-variable-test-fails` `OPEN` tester — Returned: `CRIR.Coverage.NodeKindMatrix` passes cleanly, but `CRIR.RoundTrip.Variable` FAILS. `FBlueprintEditorUtils::AddMemberVariable` is not enough to make the variable visible to the RigVM graph compile — both the source `GetMyBool` node and the round-tripped target `n0` node log `[Compiler] Variable Node ... is using a missing variable. Consider recreating the node.`, the test asserts a byte-equal round-trip, and the Result={Fail} marker fires (PDS.log lines 2509–2570 on the j_20260519T033503 run). The K2 member-variable surface and the Control Rig's RigVM external-variable registry are distinct; the fixture needs to register via the RigVM external-variable API (e.g. `URigVMController::AddExternalVariable` or whatever `ControlRigBlueprint`'s variable-add path uses) for the AddVariableNode call to bind cleanly. Test: `system.run_tests` with tests=["EditorAutomationRpcGateway.CRIR.RoundTrip.Variable"] → status=failed, error=TESTS_FAILED.
- `#4-fixture-uses-rigvm-addmembervariable` `IN-REVIEW` developer — Root cause: `URigVMController::AddVariableNode` resolves variables via `GetExternalVariablesDelegate`, which `URigVMBlueprint::HandleConfigureRigVMController` binds to read external variables off the generated-class CDO. `FBlueprintEditorUtils::AddMemberVariable` populates `NewVariables` but never regenerates the class, so the CDO has no property and the delegate returns empty → `GetVariableByName` fails → `[Compiler] Variable Node ... is using a missing variable`. The canonical path is `URigVMBlueprint::AddMemberVariable(InName, InCPPType, ...)` (engine `RigVMBlueprint.cpp:2736`), which both registers the K2 member AND runs `FBlueprintCompilationManager::CompileSynchronously` so the CDO is regenerated. Tester's hint name `URigVMController::AddExternalVariable` does not exist; the controller-side registration goes through the delegate. Fix: `Tests/Assets/TestCRIRVariableNode.cpp` — replaced the two `FBlueprintEditorUtils::AddMemberVariable(BP, VarName, BoolType)` calls with `SourceBP->AddMemberVariable(VarName, TEXT("bool"))` / `TargetBP->AddMemberVariable(VarName, TEXT("bool"))`, dropped the unused `FEdGraphPinType BoolType` block and the `EdGraphSchema_K2.h` / `Kismet2/BlueprintEditorUtils.h` includes, and refreshed the header comment to explain why the RigVM path is required. Counterfactual: removing the `URigVMVariableNode` arm in `CRIRDecompiler.cpp` ~line 692 drops the `var` line from `Result1.CRIRText`. Byte-equality alone wouldn't catch the regression (both Result1 and Result2 lose the line symmetrically), so the test also asserts `Result1.CRIRText.Contains(TEXT("var MyBool bool"))` — that assertion fails loudly if the decompile arm is removed.
- `#5-verify-fix` `DONE` tester — Verified: `system.run_tests` on j_20260521T043722_d4083794 with tests=["EditorAutomationRpcGateway.CRIR.RoundTrip.Variable"] completed with has_errors=false; PDS.log line 2306 shows `Result={Success} Name={Variable} Path={EditorAutomationRpcGateway.CRIR.RoundTrip.Variable}`. Round-trip now passes via the `URigVMBlueprint::AddMemberVariable` path; no "[Compiler] Variable Node ... is using a missing variable" warnings in the run.
- `#6-verify-full-matrix` `DONE` tester — Verified: status frontmatter still said IN-REVIEW despite #5's PASS, so re-ran the matrix end-to-end. `system.run_tests` j_20260521T050021_ec8de8a5 with tests=["EditorAutomationRpcGateway.CRIR.Coverage.NodeKindMatrix","EditorAutomationRpcGateway.CRIR.RoundTrip.Variable"] → has_errors=false, both resolved. j_20260521T050429_fb84d7d4 with tests=[Comment, Reroute, FunctionRef] → has_errors=false (the wildcard-shaped "Collapse" name doesn't exist — actuals are CollapseSingleLevel / CollapseNested). j_20260521T050509_a6c07624 with tests=[Template, DispatchArrayAdd, CollapseSingleLevel, CollapseNested, AggregateAsCollapse, InvokeEntry, Enum, IfSelect] → all 8 resolved, has_errors=false. Confirmed every matrix row from the ticket body has at least one registered `EditorAutomationRpcGateway.CRIR.RoundTrip.*` test (Unit/Hierarchy/ForwardsSolve covers `URigVMUnitNode`; Template covers bare `URigVMTemplateNode`; IfSelect covers `URigVMDispatchNode` for if+select; DispatchPrint+DispatchArrayAdd cover the other dispatch variants; Variable, Comment, Reroute, Enum, InvokeEntry, CollapseSingleLevel, CollapseNested, FunctionRef, AggregateAsCollapse cover their rows; Control.* covers all FRigControlValue variants + MaximalSettings). Coverage matrix complete, taxonomy gate active, fix verified.
