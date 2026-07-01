---
id: E-ai-assign-verbs-need-set-default-fallback
title: "`ai.assign_blackboard` new-variable path reports `propertyAssigned:false` and leaves the CDO null (missing compile, mirror the BT fix)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [ai, blackboard, ai-controller, assign, propertyAssigned, cdo, compile]
---

# `ai.assign_blackboard`'s add-variable path never writes the CDO

When wiring a Blackboard onto a freshly created AI Controller blueprint, `ai.assign_blackboard` returns `propertyAssigned:false` and leaves the CDO value null. It only registers the `DefaultBlackboard` Blueprint member-variable *definition*; the value never lands, so the agent is forced into the multi-step `blueprint.compile` + `blueprint.set_default {propertyName: "DefaultBlackboard"}` workaround to actually achieve the assignment the verb's name promises.

## Root cause

Plugin-created AI controllers parent off `AAIController`, which exposes no native `UBlackboardData*` `FObjectProperty`, so the live code path is the add-member-variable branch. That branch calls `FBlueprintEditorUtils::AddMemberVariable(...)` and then **immediately** `Controller->GeneratedClass->FindPropertyByName(VarName)` with no intervening recompile. `AddMemberVariable` only appends to `NewVariables`; the `FProperty` does not exist on `GeneratedClass` until the blueprint is recompiled. So the `CastField<FObjectProperty>` is null, `bPropertySet` stays false, the CDO write (which also targets the stale pre-add `CDO`) never runs, and the handler honestly returns `propertyAssigned:false`.

This is an asymmetry left behind by the sibling fix `B-stop-behavior-tree-noop-wrong-var` (commit 423f0ef): that commit gave `ai.assign_behavior_tree`'s add-variable branch the compile-then-fetch-fresh-CDO-then-set sequence, but did **not** apply the same fix to `ai.assign_blackboard`. `ai.assign_behavior_tree` therefore already writes the CDO and returns `propertyAssigned:true` (locked in by `FAIStopBehaviorTreeClearsAssignmentTest`, which asserts the BT is set on the CDO immediately after assign). Only the blackboard verb is still broken. (The original ticket asserted both assign verbs were no-ops and asked for a docs `set_default`-fallback overlay; the BT half was already fixed before this ticket was worked, and once the blackboard code is fixed there is no fallback to document — both claims dropped.)

## What it should do

`ai.assign_blackboard`'s add-variable branch should mirror `ai.assign_behavior_tree`'s add-variable branch (AIHandler.cpp ~441-451): after `AddMemberVariable`, call `FKismetEditorUtilities::CompileBlueprint(Controller)`, re-fetch the fresh CDO from `GeneratedClass`, re-resolve the now-materialized `DefaultBlackboard` property, and set the value on that new CDO so `bPropertySet` becomes true. Then `propertyAssigned:true` is honest and the CDO persists across reload — making the `set_default` workaround unnecessary.

**Workaround (until fixed):** after `ai.assign_blackboard`, run `blueprint.compile`, then `blueprint.set_default {path, propertyName: "DefaultBlackboard", value: "<blackboardAssetPath>"}`.

## Fix

Insert the compile-then-set block from the BT add-variable branch into the blackboard add-variable branch in `AIHandler.cpp`: `FKismetEditorUtilities::CompileBlueprint(Controller)`, fetch a fresh `AAIController* NewCDO` from `Controller->GeneratedClass->GetDefaultObject()`, `FindPropertyByName("DefaultBlackboard")`, cast to `FObjectProperty`, `SetObjectPropertyValue(...NewCDO..., BB)`, set `bPropertySet=true`. Add a regression test parallel to `FAIStopBehaviorTreeClearsAssignmentTest` that creates a controller + blackboard, calls `ai.assign_blackboard`, and reads the CDO back to assert the blackboard is set (would fail if the fix were reverted).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `ai.stop_behavior_tree` task (33 calls). PROCESS friction: `ai.assign_behavior_tree` and `ai.assign_blackboard` returned `propertyAssigned:false` and did not write the CDO even after `blueprint.compile`; the agent fell back to `blueprint.set_default` for every write, and pre-compile `property.get` on `DefaultBehaviorTree`/`DefaultBlackboard` returned `PROPERTY_NOT_FOUND` (vars don't resolve on the CDO until compiled). ~8 calls to accomplish a 2-property wire. Distinct from the judge-filed `B-stop-behavior-tree-noop-wrong-var` (stop verb, wrong variable name) — this covers the assign-side no-op and the excessive-step round trip. Docs overlay to improve: `docs/wiki-src/ai.md`.
- `#2-reword` `OPEN` developer — Rescoped to reality. The BT half of the original claim is already fixed at HEAD: commit 423f0ef (the `B-stop-behavior-tree-noop-wrong-var` fix) added `FKismetEditorUtilities::CompileBlueprint` to `ai.assign_behavior_tree`'s add-variable branch (AIHandler.cpp:441-451), and `FAIStopBehaviorTreeClearsAssignmentTest` (TestAIHandlers.cpp:174) already asserts the BT lands on the CDO after assign. The blackboard half is the genuine, uncovered remainder; the docs `set_default`-fallback ask is moot once the code is fixed (and `docs/wiki-src/ai.md` carries no such note). Retitled/reframed from "both assign verbs + docs fallback (ergonomic)" to "fix `ai.assign_blackboard`'s add-variable CDO write (bug)".
- `#3-fix` `IN-REVIEW` developer — Mirrored the BT add-variable fix into `ai.assign_blackboard`'s add-variable branch: after `FBlueprintEditorUtils::AddMemberVariable`, call `FKismetEditorUtilities::CompileBlueprint(Controller)`, re-fetch the fresh CDO, re-resolve `DefaultBlackboard`, and set the blackboard on the new CDO so `bPropertySet`/`propertyAssigned` are true and the CDO persists. File: `Source/EditorAutomationRpcGateway/Private/Handlers/AI/AIHandler.cpp` (blackboard add-variable branch, ~524-540). Regression test: `FAIAssignBlackboardWritesCDOTest` ("EditorAutomationRpcGateway.ai.assign_blackboard.WritesCDO") in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAIHandlers.cpp` — creates a controller + blackboard, calls `ai.assign_blackboard`, asserts `propertyAssigned:true` and that the first `UBlackboardData*` property on the reloaded CDO is non-null; fails if the compile-then-set fix is reverted. Note sibling `B-stop-behavior-tree-noop-wrong-var` (IN-REVIEW) touches the same file but a different branch (assign_behavior_tree / stop), so no line collision. Did not compile/run tests (later phase).
</content>
</invoke>
