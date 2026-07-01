---
id: B-stop-behavior-tree-noop-wrong-var
title: "`ai.stop_behavior_tree` is a silent no-op — removes the wrong variable and always returns stopped:true"
status: IN-REVIEW
severity: High
category: bug
tags: [ai, behavior-tree, ai-controller, silent-noop, stopped-misreport, assign-stop-mismatch]
---

# `ai.stop_behavior_tree` never clears the behavior tree assigned by `ai.assign_behavior_tree`

`ai.stop_behavior_tree` is documented as "Remove behavior tree assignment from an AI Controller" and returns `{"stopped": true}`, but it never clears the behavior tree that `ai.assign_behavior_tree` actually assigns. The two handlers operate on **different variable names**, so stop can never undo assign, yet it reports success unconditionally — a silent success-with-no-effect plus an output-field misreport (`stopped: true` when nothing was stopped).

## Root cause (AIHandler.cpp)

- `ai.assign_behavior_tree` (lines ~368–450) sets the BT either onto an existing `UBehaviorTree*` CDO property, or — when none exists — creates a **`DefaultBehaviorTree`** Blueprint member variable and writes the BT onto the CDO.
- `ai.stop_behavior_tree` (lines ~3116–3147) calls `FBlueprintEditorUtils::RemoveMemberVariable(ControllerBP, TEXT("AssignedBehaviorTree"))` — a variable named **`AssignedBehaviorTree`**, which `assign_behavior_tree` never creates. (Only the separate `ai.run_behavior_tree` alias at lines ~3062–3114 creates `AssignedBehaviorTree`.)

Because the names mismatch (`DefaultBehaviorTree` vs `AssignedBehaviorTree`), `stop_behavior_tree` removes nothing in the normal assign→stop flow. Worse, it returns `stopped: true` even when there is no `AssignedBehaviorTree` variable to remove — `RemoveMemberVariable` is a no-op on a missing variable and the handler hard-codes `stopped: true` with no check of whether anything was removed.

## Replay-confirmed repro (live editor, `mcp__editor-automation__call`)

On `/Game/AI/Controllers/BP_GuardAIController` (an `AIController` Blueprint with a `DefaultBehaviorTree` `UBehaviorTree*` member, as produced by `ai.assign_behavior_tree`):

1. Seed the value:
   `blueprint.set_default {path: "/Game/AI/Controllers/BP_GuardAIController", propertyName: "DefaultBehaviorTree", value: "/Game/AI/BehaviorTrees/BT_GuardPatrol"}`
2. Confirm it is set:
   `property.get {objectPath: ".../Default__BP_GuardAIController_C", propertyName: "DefaultBehaviorTree"}`
   → `value: "/Game/AI/BehaviorTrees/BT_GuardPatrol.BT_GuardPatrol"`
3. Call the method under test:
   `ai.stop_behavior_tree {controllerPath: "/Game/AI/Controllers/BP_GuardAIController"}`
   → `{"controllerPath": "/Game/AI/Controllers/BP_GuardAIController", "stopped": true}`
4. Read back:
   `property.get {objectPath: ".../Default__BP_GuardAIController_C", propertyName: "DefaultBehaviorTree"}`
   → `value: "/Game/AI/BehaviorTrees/BT_GuardPatrol.BT_GuardPatrol"`  **(still set — stop was a no-op on the CDO)**

The `blueprint.inspect` variable list at the time showed only `DefaultBlackboard` and `DefaultBehaviorTree` — there is no `AssignedBehaviorTree` variable for stop to remove, yet it still claimed `stopped: true`.

## Impact

The natural "set up an AI brain, then later remove the behavior tree" workflow (assign_behavior_tree → … → stop_behavior_tree) silently fails: the controller keeps auto-pointing at the behavior tree, but the caller is told `stopped: true`. A fuzz attempt agent doing exactly this had to abandon `ai.stop_behavior_tree` and fall back to `blueprint.set_default {propertyName: "DefaultBehaviorTree", value: ""}` to actually clear the assignment. There is no way for a JSON-RPC caller to detect the no-op from the response.

**Workaround:** clear the assignment with `blueprint.set_default {propertyName: "DefaultBehaviorTree", value: ""}` (note: `value: "None"` errors with `CONVERSION_FAILED` — use the empty string).

**Fix options:**
1. Make `stop_behavior_tree` clear whatever `assign_behavior_tree` set: null the existing `UBehaviorTree*` CDO property (the same property the assign handler discovers/creates, i.e. `DefaultBehaviorTree`) rather than removing a `AssignedBehaviorTree` member by name. Mirror the assign handler's property-discovery loop. Then re-mark/save and recompile so the CDO null persists.
2. At minimum, stop hard-coding `stopped: true`: report `stopped: false` (or `cleared: false`) when nothing was actually removed/nulled, so the contract is honest. Also reconcile the `AssignedBehaviorTree` (run alias) vs `DefaultBehaviorTree` (assign) naming so stop covers both paths.

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode fuzz of `ai.stop_behavior_tree`. Replay-confirmed on `/Game/AI/Controllers/BP_GuardAIController`: seeded `DefaultBehaviorTree=/Game/AI/BehaviorTrees/BT_GuardPatrol` via `blueprint.set_default`, verified via `property.get`, then `ai.stop_behavior_tree {controllerPath}` returned `stopped: true` while a follow-up `property.get` showed `DefaultBehaviorTree` still `BT_GuardPatrol`. Root cause read from `AIHandler.cpp`: `stop_behavior_tree` removes a member variable `AssignedBehaviorTree`, but `assign_behavior_tree` assigns onto/creates `DefaultBehaviorTree` — name mismatch makes stop a no-op, and it hard-codes `stopped: true` regardless. Not a duplicate of the DONE silent-success tickets (`B-material-stub-handlers-silent-success`, `B-blueprint-remove-function-silent-noop-on-custom-event`), which cover other namespaces.
- `#2-fix` `IN-REVIEW` developer — Implemented Fix option 1 (root cause) + option 2 (honest contract) in `Source/EditorAutomationRpcGateway/Private/Handlers/AI/AIHandler.cpp` `ai.stop_behavior_tree`. Stop now mirrors `assign_behavior_tree`'s property-discovery loop: it finds the first `UBehaviorTree*` property on the controller CDO (the discovered native property / the `DefaultBehaviorTree` member var assign writes) and nulls it via reflection, then marks-modified + saves. It also still removes the legacy `AssignedBehaviorTree` member-var definition (the `run_behavior_tree` alias path) but only after `FindNewVariableIndex` confirms it exists. `stopped` is now set from `bCleared` (true only when a BT value was actually nulled or a var removed) instead of being hard-coded `true`, and a `clearedProperty` field is added on success — fixing both the silent no-op and the output misreport. Regression test: `EditorAutomationRpcGateway.ai.stop_behavior_tree.ClearsAssignedTree` in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAIHandlers.cpp` (`FAIStopBehaviorTreeClearsAssignmentTest`) drives the real assign→stop flow (create AI controller → `behavior_tree.create` → `ai.assign_behavior_tree` → assert BT set on CDO → `ai.stop_behavior_tree` → assert `stopped:true` and BT nulled on CDO). It fails against the reverted code: the old handler removed only `AssignedBehaviorTree`, leaving the BT set on the CDO, while still reporting `stopped:true`.
