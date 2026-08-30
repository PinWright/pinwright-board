---
id: B-run-behavior-tree-silent-noop
title: "`ai.run_behavior_tree` is a silent no-op — documented as an alias for `assign_behavior_tree` but never sets `DefaultBehaviorTree`, returns `assigned:true` regardless"
status: IN-REVIEW
severity: High
category: bug
tags: [ai, behavior-tree, ai-controller, silent-noop, alias-divergence, assign-misreport]
---

# `ai.run_behavior_tree` registers the BT nowhere usable yet reports `assigned:true`

`ai.run_behavior_tree` is documented (method summary, verbatim) as **"Assign a
Behavior Tree to run on an AI Controller (alias for assign_behavior_tree)"**, but
its implementation diverges completely from `ai.assign_behavior_tree`. It never
sets the `DefaultBehaviorTree` CDO property (the property the engine reads to
auto-run a BT, and exactly what `assign_behavior_tree` sets), and it never sets
the value of the variable it does touch. It instead adds an **empty, valueless**
`AssignedBehaviorTree` member variable, marks the blueprint modified, saves, and
returns `{"assigned": true}` unconditionally. The BT ends up wired nowhere, so a
caller who follows the documented alias contract gets a silent
success-with-no-effect.

## Root cause (AIHandler.cpp)

- `ai.assign_behavior_tree` (lines ~438–487) calls
  `AssignObjectDefaultToControllerCDO(Controller, UBehaviorTree::StaticClass(),
  BT, TEXT("DefaultBehaviorTree"), PropertyName)` — it writes the BT onto the
  controller CDO's `DefaultBehaviorTree` property (creating it if needed) and
  returns `propertyAssigned`. This is the correct, engine-readable wiring.
- `ai.run_behavior_tree` (lines ~3239–3291), despite the "alias for
  assign_behavior_tree" summary, does **only**:
  ```cpp
  FBlueprintEditorUtils::AddMemberVariable(ControllerBP, TEXT("AssignedBehaviorTree"), PinType);
  FBlueprintEditorUtils::MarkBlueprintAsModified(ControllerBP);
  McpSafeAssetSave(ControllerBP);
  // ... SetBoolField("assigned", true)
  ```
  It adds an empty `UBehaviorTree*` member variable named `AssignedBehaviorTree`,
  **never sets that variable's value to `BT`**, and **never touches
  `DefaultBehaviorTree`** — the only property the engine (and
  `assign_behavior_tree`) uses. The `BT` it loaded is used only for the
  existence check; its reference is discarded. `assigned` is hard-coded `true`.

So the documented alias does not assign the behavior tree at all — neither the
default-BT property the engine runs, nor a populated variable.

## Replay-confirmed repro (live editor, `mcp__editor-automation__call`)

On fresh isolated assets (created + verified absent, then deleted):

1. `ai.create_behavior_tree {name:BT_OracleRun, path:/Game/AIOracleRun}` → created
2. `ai.create_ai_controller {name:BP_OracleRunController, path:/Game/AIOracleRun}` → created
3. Baseline — confirm the controller has no default BT yet:
   `property.get {objectPath:/Game/AIOracleRun/BP_OracleRunController.Default__BP_OracleRunController_C, propertyName:DefaultBehaviorTree}`
   → `[PROPERTY_NOT_FOUND] ... Property 'DefaultBehaviorTree' not found`
4. Method under test:
   `ai.run_behavior_tree {controllerPath:/Game/AIOracleRun/BP_OracleRunController, behaviorTreePath:/Game/AIOracleRun/BT_OracleRun}`
   → `{"controllerPath":"/Game/AIOracleRun/BP_OracleRunController","behaviorTreePath":"/Game/AIOracleRun/BT_OracleRun","assigned":true}`
5. Read back the engine-run property — STILL absent (no-op):
   `property.get {... propertyName:DefaultBehaviorTree}`
   → `[PROPERTY_NOT_FOUND] ... Property 'DefaultBehaviorTree' not found`
6. Read back the variable it claims to create — also absent on the CDO:
   `property.get {... propertyName:AssignedBehaviorTree}`
   → `[PROPERTY_NOT_FOUND] ... Property 'AssignedBehaviorTree' not found`
   (the new member var is added to the blueprint but the asset is saved without a
   recompile, so it never materializes on the generated-class CDO — and even if
   it did, its value was never set to the BT.)

Contrast — the SAME task's `assign_behavior_tree` on a sibling controller DID
wire it: `property.get {... propertyName:DefaultBehaviorTree}` on
`Default__BP_GuardController_C` returns `value:"/Game/AI/BT_Guard.BT_Guard"`.

## Impact

The natural "run this behavior tree on startup" phrasing maps directly to
`ai.run_behavior_tree` (the verb is literally named for it and documented as the
assign alias). A caller who uses it is told `assigned:true` while the BT is wired
nowhere the engine reads — the controller will not run the tree. There is no
signal in the response to detect the no-op. In this fuzz task the attempt agent
trusted `run_behavior_tree`, then discovered via `property.get` that
`DefaultBehaviorTree` was missing and had to fall back to `ai.assign_behavior_tree`
to actually wire the BT.

This is distinct from `B-stop-behavior-tree-noop-wrong-var` (IN-REVIEW), which is
about `ai.stop_behavior_tree` removing the wrong variable; that ticket only notes
in passing that `run_behavior_tree` creates the `AssignedBehaviorTree` var. The
bug HERE is that `run_behavior_tree` itself does not assign the BT despite its
"alias for assign_behavior_tree" contract and `assigned:true` response.

## Fix options

1. **Make it a real alias** (matches the documented contract): have
   `ai.run_behavior_tree` call the same path as `ai.assign_behavior_tree` —
   `AssignObjectDefaultToControllerCDO(..., TEXT("DefaultBehaviorTree"), ...)` —
   and report `assigned` from the actual `propertyAssigned` result. Drop the
   empty `AssignedBehaviorTree` member-variable creation (or, if a populated
   variable is intended, set its default value to `BT` and recompile so it
   materializes on the CDO). Reconcile with `stop_behavior_tree`
   (`B-stop-behavior-tree-noop-wrong-var`) so assign/run/stop all agree on one
   property name.
2. **At minimum, stop misreporting:** do not hard-code `assigned:true`; report
   the actual outcome (whether `DefaultBehaviorTree` was set), so a JSON-RPC
   caller can detect the no-op. If the divergence is intentional, the summary
   must stop calling it an "alias for assign_behavior_tree".

## History
- `#2-fix` `IN-REVIEW` developer — Implemented Fix option 1 (real alias). `ai.run_behavior_tree` (AIHandler.cpp:3285-3337) now routes through the same shared `AssignObjectDefaultToControllerCDO(ControllerBP, UBehaviorTree::StaticClass(), BT, TEXT("DefaultBehaviorTree"), PropertyName)` path as `ai.assign_behavior_tree`: it writes the BT onto the controller CDO's `DefaultBehaviorTree` property (compiling + creating the member var if no native property exists), echoes `propertyName`, and reports `assigned` from the actual `bAssigned` result instead of hard-coding `true`. Dropped the old empty/valueless `AssignedBehaviorTree` `AddMemberVariable` call (the silent no-op) and switched the save to `MarkBlueprintAsStructurallyModified` to match assign. This makes run a true alias on the one property name `DefaultBehaviorTree`, consistent with `assign_behavior_tree` and the stop handler's CDO-null discovery loop (B-stop-behavior-tree-noop-wrong-var); stop's legacy `AssignedBehaviorTree` member-var removal stays as a no-op safety net for pre-existing assets. Regression test added: `EditorAutomationRpcGateway.ai.run_behavior_tree.WritesCDO` (`FAIRunBehaviorTreeWritesCDOTest`) in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAIHandlers.cpp` — creates a real controller + BT, calls `ai.run_behavior_tree`, asserts the response reports `assigned:true` AND reads the CDO back via `ReadCDOObjectProp(..., UBehaviorTree::StaticClass())` to confirm the BT is actually wired onto the CDO; fails (CDO null / assigned hard-coded over a no-op) if the fix were reverted. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/AI/AIHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAIHandlers.cpp`.
- `#1-initial-repro` `OPEN` reporter — Realism-mode fuzz, guard-AI wiring task. Replay-confirmed at this HEAD on fresh isolated assets (`/Game/AIOracleRun/BP_OracleRunController` + `BT_OracleRun`): baseline `property.get DefaultBehaviorTree` → PROPERTY_NOT_FOUND; `ai.run_behavior_tree` returned `assigned:true`; follow-up `property.get` showed `DefaultBehaviorTree` STILL absent and `AssignedBehaviorTree` also absent on the CDO — a complete silent no-op. Contrasted with the same task's `ai.assign_behavior_tree`, which set `DefaultBehaviorTree=/Game/AI/BT_Guard.BT_Guard` on `Default__BP_GuardController_C`. Root cause read from `AIHandler.cpp`: `run_behavior_tree` (~3239–3291) only calls `AddMemberVariable(ControllerBP, "AssignedBehaviorTree", PinType)` (empty, value never set) and never touches `DefaultBehaviorTree`, then hard-codes `assigned:true`; whereas `assign_behavior_tree` (~438–487) writes the BT onto the CDO `DefaultBehaviorTree` via `AssignObjectDefaultToControllerCDO`. The method summary documents `run_behavior_tree` as "alias for assign_behavior_tree" — the implementation diverges. Not a duplicate of `B-stop-behavior-tree-noop-wrong-var` (that ticket is about `stop_behavior_tree`; it references `run_behavior_tree` only incidentally). Cleaned up the replay assets via `asset.delete`.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 3 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
