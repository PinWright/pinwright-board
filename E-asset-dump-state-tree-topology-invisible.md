---
id: E-asset-dump-state-tree-topology-invisible
title: "StateTree dumps lack states/transitions/tasks — properties.json is just an opaque EditorData object pointer (no state_tree.json sidecar)"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [asset-dump, state-tree, sidecar, coverage, editor-data, readback]
---

# `asset.dump` on a UStateTree emits only an opaque `EditorData` pointer — the authored topology is invisible

`asset.dump` on a `UStateTree` writes only `meta.json` and a tiny
`properties.json` whose entire useful content is a single opaque `EditorData`
object-pointer string. Everything an agent would dump a StateTree to review —
its states, parent/child topology, transitions, transition triggers, tasks,
enter conditions, evaluators, and property bindings — lives inside the
`UStateTreeEditorData` subobject (`SubTrees`, each `FStateTreeStateLink`'s
`Children` / `Tasks` / `Transitions` / `EnterConditions`, plus `Evaluators`,
`GlobalTasks`, `EditorBindings`). None of it survives the dump. The dumped
`properties.json` for a freshly authored `ST_GuardAI` (Component schema,
Root→{Patrol, Investigate, Chase}, three transitions, an `OnEvent` trigger
on each of two transitions, a `PatrolTask`, and an enter condition on
Investigate) is in its entirety:

```json
{
	"EditorData": {
		"flags": ["EditorOnly"],
		"is_overridden_locally": true,
		"type": "TObjectPtr<UObject>",
		"value": "/Game/AI/StateTrees/ST_GuardAI.ST_GuardAI:EditorData"
	}
}
```

That is the same opaque-`EditorData`-pointer shape already fixed for
`UserDefinedStruct` in **[E-asset-dump-userdefinedstruct-field-list](E-asset-dump-userdefinedstruct-field-list.md)**
(DONE) — there the resolution was a dedicated `user_defined_struct.json`
sidecar that walks the editor-data structure and emits the field list. A
StateTree needs the analogous `state_tree.json` sidecar; today there is none,
so the documented "dump the asset to review the authored structure" path
returns nothing reviewable for the asset's defining content.

This is **not** the `B-asset-dump-instanced-subobjects-not-recursed` (DONE)
recursion case: that fix recurses only properties carrying
`CPF_PersistentInstance | CPF_InstancedReference`, and `StateTree::EditorData`
is a plain `TObjectPtr<UObject>` flagged `["EditorOnly"]` (not instanced), so
the recursion correctly does not fire on it — it needs its own per-asset-class
sidecar, exactly as UserDefinedStruct did.

## Verification readbacks also dead-end (why the sidecar matters)

After the empty dump, the natural fallbacks an agent reaches for to confirm the
authored topology also fail or come up short:

- `property.get { objectPath:"…ST_GuardAI:EditorData", propertyName:"SubTrees.0.Children" }`
  → `[PROPERTY_NOT_FOUND] Failed to resolve property 'SubTrees.0.Children' …
  Cannot traverse into property 'SubTrees' of type 'ArrayProperty'` — array-index
  segments are unsupported (the generic gap tracked in
  [E-property-path-array-index-unsupported](E-property-path-array-index-unsupported.md)).
- `system.inspect.inspect_object { objectPath:"…ST_GuardAI:EditorData" }` returns
  `SubTrees` serialized only **one level deep**: each element shows `ID`, `Name`,
  and `Parameters`, but **not** the state's `Children`, `Tasks`, `Transitions`,
  or `EnterConditions`. So even the live inspector shows only the Root state's
  name, never the Patrol/Investigate/Chase children, the transitions, or the
  tasks/conditions that were authored.
- The only way to actually read the authored states/transitions/tasks is to call
  `system.inspect.inspect_object` on the engine's **auto-named** Root-state
  subobject `…ST_GuardAI.ST_GuardAI:EditorData.StateTreeState_0` — a name the
  agent had to *guess* (its first guesses `EditorData.Root` and
  `EditorData.Patrol` both returned `[OBJECT_NOT_FOUND]`). Nothing in the dump or
  the `EditorData` inspect tells you that auto-name.

So the documented dump path does not confirm any of the success-check facts for a
StateTree; the only working route is guessing an undocumented auto-subobject name.

## What it should do / how to fix

**Sidecar (mirrors the UserDefinedStruct fix), via the JSON-sidecar registry:**
emit a `state_tree.json` sidecar. Per-class JSON dumps are no longer inline
`if (Cast<T>)` branches in `AssetDumpHandler.cpp::BuildAllFilesForAsset` — the
**E-asset-dump-registry-driven-dispatch** (DONE) refactor moved them to
`REGISTER_DUMP_JSON_SIDECAR` records living next to each builder, walked by
`RunRegisteredJsonSidecars` (see the comment at `BuildAllFilesForAsset`'s generic
`else` branch). So the fix is a **new `Handlers/Asset/StateTreeDumpBuilder.cpp`**
with one `REGISTER_DUMP_JSON_SIDECAR(TEXT("state_tree"), DumpFileNames::StateTree,
…)` record keyed on `UStateTree::StaticClass()`, mirroring
`UserDefinedStructDumpBuilder.cpp` / `SoundCueDumpBuilder.cpp` — **not** a new
inline branch in `AssetDumpHandler.cpp`.

The builder walks the `UStateTreeEditorData` (`Cast<UStateTreeEditorData>(StateTree->EditorData)`)
and serializes the authored tree: the state hierarchy (`SubTrees` → each state's
`Name`/`ID`/`Type`, recursively into `Children`), per-state `Tasks`,
`EnterConditions`, and `SingleTask` (each an `FStateTreeEditorNode` whose
`Node`/`Instance` are `FInstancedStruct`s — serialize the inner script struct via
`GetScriptStruct()`/`GetMemory()`), per-state `Transitions` (target state link,
`Trigger`, required event tag for `OnEvent`, gating `Conditions`), plus tree-scoped
`Evaluators` and `GlobalTasks`. Gate the whole TU on the same
`__has_include`/`MCP_STATE_TREE_*` guard the StateTree authoring handlers use,
since `StateTreeModule`/`StateTreeEditorModule` are conditional modules
(`Build.cs` `TryAddConditionalModule`).

**Still required (the ticket gets this right):** registered JSON sidecars are NOT
auto-pulled into the diff baseline — only `IrSidecarRegistry` specs are auto-added
at `LoadBaselineDumpFiles`. So add the `DumpFileNames::StateTree` constant
(`AssetDumpHandler.h`) and list it by hand in the `FixedCanonical[]` table in
`AssetDumpHandler.cpp::LoadBaselineDumpFiles`, exactly as `user_defined_struct.json`
/ `sound_cue.json` are. (No aspect-version bump: this is a brand-new aspect, and
the cache treats a missing entry as "must regenerate".)

**Workaround (today):** `asset.dump` is not enough; to read the authored topology
call `system.inspect.inspect_object` on the auto-named Root-state subobject
`…<Asset>:EditorData.StateTreeState_0` (the first auto-numbered state subobject),
then walk its children. The name is undocumented and must be discovered by trial
(`asset.dump` will not reveal it; `EditorData.Root`/`EditorData.<StateName>` do
**not** resolve).

## History
- `#1-initial-repro` `OPEN` reporter — Struggle audit of a from-scratch StateTree authoring task (seed `state_tree.set_transition_trigger`, outcome clean: the asset `/Game/AI/StateTrees/ST_GuardAI` was authored end-to-end successfully). The verification step exposed a dump-coverage gap, confirmed by replay: `asset.dump { assetPath:"/Game/AI/StateTrees/ST_GuardAI" }` wrote only `meta.json` + `properties.json`, and the entire `properties.json` is a single opaque `EditorData` object pointer (`"value":"…ST_GuardAI:EditorData"`) — no states, transitions, triggers, tasks, conditions. The authored topology all lives inside the `UStateTreeEditorData` subobject (`SubTrees` etc.) and is invisible in the dump; there is no `state_tree.json` sidecar. Same opaque-`EditorData`-pointer shape as the DONE `E-asset-dump-userdefinedstruct-field-list` (UserDefinedStruct), which was resolved with a dedicated `user_defined_struct.json` sidecar — StateTree needs the analogous `state_tree.json`. Distinct from the instanced-subobject recursion fix `B-asset-dump-instanced-subobjects-not-recursed` (DONE): `EditorData` is a plain `TObjectPtr<UObject>` flagged `["EditorOnly"]`, not `CPF_PersistentInstance`/`CPF_InstancedReference`, so that recursion correctly does not fire — a per-class sidecar is required. Replay-confirmed the readback dead-ends too: `property.get SubTrees.0.Children` → `[PROPERTY_NOT_FOUND] … Cannot traverse into property 'SubTrees' of type 'ArrayProperty'` (the array-index gap in `E-property-path-array-index-unsupported`), and `system.inspect.inspect_object` on `…:EditorData` serializes `SubTrees` only one level deep (Root state's `Name`/`ID`/`Parameters`, never its `Children`/`Tasks`/`Transitions`/`EnterConditions`). The only working route was `inspect_object` on the auto-named `…:EditorData.StateTreeState_0` Root subobject, a name the attempt had to guess (`EditorData.Root` and `EditorData.Patrol` both `[OBJECT_NOT_FOUND]`). No existing board ticket on StateTree dump coverage (ripgrep across OPEN + closed; qmd unavailable). Proposes a `state_tree.json` sidecar in `Handlers/Asset/AssetDumpHandler.cpp` mirroring the UDS builder, serializing state hierarchy + per-state tasks/conditions/transitions(+trigger/event tag) + tree-scoped evaluators/global-tasks/bindings via `Utils/PropertyUtils`.
- `#2-reword+sidecar` `IN-REVIEW` developer — Reworded the **Fix** to match the DONE `E-asset-dump-registry-driven-dispatch` refactor: per-class JSON dumps are no longer inline `if (Cast<T>)` branches in `AssetDumpHandler.cpp::BuildAllFilesForAsset` but `REGISTER_DUMP_JSON_SIDECAR` records walked by `RunRegisteredJsonSidecars`. Implemented as a new registry sidecar, not an inline branch. **Changed:** added `Handlers/Asset/StateTreeDumpBuilder.{h,cpp}` — a `REGISTER_DUMP_JSON_SIDECAR(TEXT("state_tree"), DumpFileNames::StateTree, …)` keyed on `UStateTree::StaticClass()` that walks `UStateTreeEditorData` and serializes the authored topology: `subTrees` → each state's `name`/`id`/`type`/`selectionBehavior`/`tag`, recursively into `children`; per-state `enterConditions`/`tasks`/`singleTask` (each `FStateTreeEditorNode` with its `Node`/`Instance` `FInstancedStruct` expanded to inner-struct fields via `GetScriptStruct()`/`GetMemory()` + `FJsonObjectConverter::UStructToJsonObject`, tagged `_kind`); per-state `transitions` (target state link name/id/linkType, symbolic `trigger`, `requiredEventTag`/`requiredEventPayloadStruct` for `OnEvent`, gating `conditions`); plus tree-scoped `evaluators` and `globalTasks`. Added `DumpFileNames::StateTree = "state_tree.json"` (`AssetDumpHandler.h`) and listed it in the `FixedCanonical[]` baseline table (`AssetDumpHandler.cpp::LoadBaselineDumpFiles`) — registered JSON sidecars are NOT auto-added there (only `IrSidecarRegistry` specs are). The whole TU is gated on the same `__has_include("StateTree.h"/…)` guard the StateTree authoring handlers use (conditional `StateTreeModule`/`StateTreeEditorModule`); when absent the sidecar's class thunk returns nullptr so `RunRegisteredJsonSidecars` never matches it. No aspect-version bump (brand-new aspect → cache treats missing entry as must-regenerate). **Test:** `Tests/Assets/TestStateTreeDumpBuilder.cpp` (two `IMPLEMENT_SIMPLE_AUTOMATION_TEST`s, identically `__has_include`-gated) authors a transient `UStateTree` (Root subtree → `Patrol` child → Root→Patrol `OnStateCompleted` transition → a task node whose inner struct is resolved by reflection) and asserts (a) `BuildStateTreeJson` surfaces `subTrees[Root].children[Patrol]`, the transition's `trigger`/`target.name`, and the task's expanded `nodeType`/`node._kind`; and (b) `AssetDumpHandler::DumpSingleAsset` writes a parseable `state_tree.json` with a non-empty `subTrees`. Both would fail if the sidecar were reverted (today only `meta.json` + the opaque-pointer `properties.json` are written).
- `#3-additional-sidecar-omits-editor-bindings` `IN-REVIEW` reporter — Additional evidence (sidecar now lands, but property bindings still invisible): re-ran the from-scratch StateTree authoring + verify task (seed `state_tree.bind_property`; the authoring agent reported friction "the named state_tree.json sidecar does NOT serialize editor property bindings — it only shows unresolved per-node bindingsBatch handles (65535), so success-check criterion (c) cannot be verified from the dump"). Replay-confirmed on a fresh `/Game/AI/StateTrees/ST_OracleRepro` (Component schema): `ai.create_state_tree` → `ai.add_state_tree_state Investigate under Root` → `state_tree.add_evaluator FTestEval_A name=SightEval` (evaluatorCount:1) → `state_tree.add_task FStateTreeDelayTask name=InvestigateTask on Investigate` (taskCount:1) → `state_tree.bind_property {sourcePath:"SightEval.FloatA", targetPath:"InvestigateTask.Duration", save:true}` → `{"sourcePath":"FloatA","targetPath":"Duration","bindingCount":1,"saved":true}`. The `#2` sidecar fix IS now in the running plugin: `asset.dump {assetPath:"/Game/AI/StateTrees/ST_OracleRepro"}` now writes `state_tree.json` (alongside `meta.json`+`properties.json`) and it correctly surfaces `evaluators[SightEval]` (nodeType `FTestEval_A`, instance `floatA`), `subTrees[Root].children[Investigate]`, and `tasks[InvestigateTask]` (nodeType `FStateTreeDelayTask`, instance `duration`). **But the authored editor property binding is entirely absent from the sidecar.** There is NO `editorBindings`/`bindings` array; the only binding-related fields in `state_tree.json` are the per-node runtime `bindingsBatch`/`outputBindingsBatch` handles, every one set to the uncompiled sentinel `{"value":65535}` (verbatim, on both the SightEval node and the InvestigateTask node). So the dump shows the FloatA→Duration binding nowhere. Confirmed the binding really persisted (so this is a dump-coverage gap, not a bind failure): `property.get {objectPath:"…ST_OracleRepro.ST_OracleRepro:EditorData", propertyName:"EditorBindings"}` returns `PropertyBindings[0]` with `SourcePropertyPath.Segments[0].Name:"FloatA"` and `TargetPropertyPath.Segments[0].Name:"Duration"`. So success-check criterion (c) "confirm the new binding from SightEval to InvestigateTask is recorded" still cannot be answered from `asset.dump`; the only working route remains `property.get EditorData.EditorBindings`. The `#2` implementation serialized state hierarchy + per-state tasks/conditions/transitions + tree-scoped evaluators/global-tasks but **omitted `UStateTreeEditorData::EditorBindings` (`FStateTreeEditorPropertyBindings::PropertyBindings`)** — the original ticket body lists `EditorBindings` among the editor-data content the dump must surface, so this is a not-yet-complete part of the same fix, not a new ticket. The tester verifying `#2` should require the sidecar to emit an `editorBindings` array (each entry's `SourcePropertyPath`/`TargetPropertyPath` segment names, e.g. `FloatA`→`Duration`) before flipping DONE; the uncompiled `bindingsBatch:65535` handles are not a substitute (they are only resolved after compile and never name the source/target property). No separate board ticket covers state-tree binding dump coverage (ripgrep across OPEN+closed for `EditorBindings|PropertyBindings|bindingsBatch`; qmd unavailable).
