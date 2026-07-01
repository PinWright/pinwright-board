---
id: B-merge-actors-rejects-valid-selection
title: "performance.merge_actors can never complete headless (wrong documented toolName + modal save dialog in RunMergeFromSelection)"
status: IN-REVIEW
severity: High
category: bug
tags: [performance, merge_actors, mesh-merging, selection, draw-calls]
---

# performance.merge_actors can never complete headless

`performance.merge_actors` is non-functional for its primary, advertised use
case ("the standard workflow for reducing draw calls on grouped static
geometry"). It cannot merge two valid `StaticMeshActor`s that each carry a
`UStaticMeshComponent` with a non-null assigned static mesh — the exact workflow
the tool description advertises. Two confirmed root causes block it, one of them
unconditionally:

1. **The documented `toolName` is wrong**, so the documented call variant never
   name-matches a registered tool.
2. **The chosen engine API (`RunMergeFromSelection()`) always pops a blocking
   modal save dialog**, which cannot be answered in a headless MCP context, so
   the merge can never run to completion regardless of selection.

Reproduced live against `ExampleProjectWelcome`, which contains two
StaticMeshActors (`UELogo`, `UELogo2`) both carrying a `UStaticMeshComponent`
with a non-null `/Game/Global/DemoRoom/Meshes/UELogo.UELogo` mesh.

**Repro:**

- `performance.merge_actors {actors:["UELogo","UELogo2"], toolName:"MeshMerging"}`
  -> `[MERGE_TOOL_UNAVAILABLE] No Merge Actors tool can operate on the current selection`
- `performance.merge_actors {actors:["UELogo","UELogo2"]}` (auto-pick)
  -> `[MERGE_TOOL_UNAVAILABLE]`

## Confirmed root causes

**1. The documented `toolName` is wrong.** The handler matches the requested
`toolName` against `IMergeActorsTool::GetToolNameText().ToString()`
(`PerformanceHandler.cpp`, the `RequestedToolName` loop). In UE 5.7
`FMeshMergingTool::GetToolNameText()` returns the literal **"Merge"**
(`Engine/Source/Editor/MergeActors/Private/MeshMergingTool/MeshMergingTool.cpp:57`,
`LOCTEXT("MeshMergingToolName", "Merge")`), **not** "MeshMerging". So the
documented default and example (`toolName` "defaults to MeshMerging",
"e.g. 'MeshMerging', 'MeshProxy'") never name-match any registered tool — the
name lookup silently misses and falls through to the `CanMergeFromSelection()`
auto-pick. (There is no `merge_actors` wiki-src overlay; this param-spec string
IS the documented value.)

**2. (unconditional headless blocker) modal save dialog in
`RunMergeFromSelection()`.** Even with the correct tool and a valid selection,
`RunMergeFromSelection()` -> `GetPackageNameForMergeAction(GetDefaultPackageName(), ...)`.
`FMeshMergingTool::GetDefaultPackageName()` ALWAYS returns a non-empty
`MERGED_<ActorName>` path (`MeshMergingTool.cpp:65-89`). With a non-empty
default, `GetPackageNameForMergeAction` ALWAYS calls
`ContentBrowserModule.CreateModalSaveAssetDialog(...)`
(`MergeProxyUtils/Utils.cpp:93-119`) — a blocking modal "Create Merged Actor"
dialog that cannot be answered headlessly. So `RunMergeFromSelection()` is
simply the wrong API for a headless RPC; it can never complete. The handler
supplies no explicit output package to bypass the dialog.

## Not a defect: the selection hand-off (earlier hypothesis, retracted)

An earlier revision of this ticket asserted that `CanMergeFromSelection()`
returns false because the handler's selection "is not seen by the merge module"
(a "typed-element vs legacy USelection" split, fixable with
`NoteSelectionChange`). **Engine source falsifies that:**
`CanMergeFromSelection()` -> `BuildMergeComponentDataFromSelection()` reads
exactly `GEditor->GetSelectedActors()` (`MergeProxyUtils/Utils.cpp:22-41`,
`MergeActorsTool.cpp:66-71`) — the SAME classic `USelection` set the handler
populates with `GEditor->SelectActor(Actor, true, /*bNotify*/true, true)`, which
routes through that selection's `UTypedElementSelectionSet` and notifies
synchronously (`EditorSelectUtils.cpp:604-643`). There is no separate selection
source and no `NoteSelectionChange` gating the read; `FMeshMergingTool` does not
override the base `CanMergeFromSelection`. `NoteSelectionChange` only fires
delegates; it does not change the set's contents, so it would not flip the
result. The `MERGE_NOT_POSSIBLE` symptom for the `toolName:"Merge"` variant was
not independently reproduced (the prior repro could not confirm the editor
selection set held the two actors at merge time — `actor.select`'s
`selectedCount` reports the intended count, not a read-back). The
selection-tweak / `NoteSelectionChange` step is therefore dropped from the fix,
which stops relying on the editor selection / tool gate and drives the merge
directly.

**Impact:** the entire `performance.merge_actors` capability is non-functional;
an agent cannot bake clustered static geometry into a single draw via this
tool, and there is no sibling RPC that performs an actor merge.

**Workaround:** none via this RPC.

**Fix:**
- Correct the `toolName` doc to the actual tool name ("Merge"), and case/alias-map
  "MeshMerging" -> the Merge tool so the historically-documented value also works;
  reject any other tool name with `MERGE_TOOL_UNAVAILABLE` rather than silently
  running a mesh merge under a name the handler does not perform.
- Stop routing through the interactive Merge Actors tool registry
  (`IMergeActorsModule` / `IMergeActorsTool::Run/CanMergeFromSelection`). Replace
  `RunMergeFromSelection()` with a headless merge that takes an explicit output
  package (bypassing `CreateModalSaveAssetDialog`): resolve the requested actors
  directly, collect their `UStaticMeshComponent`s with a non-null
  `GetStaticMesh()`, and call `IMeshMergeUtilities::MergeComponentsToStaticMesh`
  (`MeshMergeModule.GetUtilities()`) with a computed, non-empty output package
  (caller-supplyable via a new `outputPackage` param, default a unique
  `/Game/Merged/MERGED_<FirstActor>`) so no modal dialog is ever reached.
- Register the produced assets, spawn the merged `AStaticMeshActor` referencing
  the produced `UStaticMesh` at the merge location, and when `replaceSourceActors`
  is true destroy the source actors. Requires linking the `MeshMergeUtilities`
  module. Return the actual merged package path (`mergedPackageName` /
  `outputPackage` / `mergedAssetPath`); error with the existing `MERGE_*` codes
  when there is no static mesh to merge or the merge produces no asset.

## History
- `#1-initial-repro` `OPEN` reporter — Filed from the test workflow (seed `performance.merge_actors`). Replay-confirmed live against `ExampleProjectWelcome` with two valid StaticMeshActors (`UELogo`, `UELogo2`, both with non-null `/Game/Global/DemoRoom/Meshes/UELogo.UELogo`): all three documented call variants fail (`MERGE_TOOL_UNAVAILABLE` for toolName `MeshMerging`/auto, `MERGE_NOT_POSSIBLE` for the correct toolName `Merge`), including after an explicit `actor.select` reporting `selectedCount:2`. Root-caused to (1) wiki `toolName` "MeshMerging" never name-matching the engine's `GetToolNameText()=="Merge"`, (2) `CanMergeFromSelection()` returning false despite a valid editor selection (selection hand-off not seen by the merge module), and (3) a latent modal save dialog in `GetPackageNameForMergeAction` that would block a headless merge even if the selection passed.
- `#2-reword` `OPEN` developer — Reworded after engine-source review. Defect #2's stated root cause (typed-element-vs-USelection split; `NoteSelectionChange` fix) is wrong: `CanMergeFromSelection()` (`MergeActorsTool.cpp:66-71`) reads the SAME `GEditor->GetSelectedActors()` the handler writes (`MergeProxyUtils/Utils.cpp:22-41`; `EditorSelectUtils.cpp:604-643`). The genuine structural blocker is the modal save dialog inside the interactive tool path (`RunMergeFromSelection` -> `GetPackageNameForMergeAction` -> `CreateModalSaveAssetDialog`, `Utils.cpp:93-119`), which is unanswerable headless. Defect #1 (misdocumented `toolName` "MeshMerging" vs engine `"Merge"`) confirmed. Title/severity/**Fix:** rewritten to drive `IMeshMergeUtilities::MergeComponentsToStaticMesh` directly with a computed package + spawn the merged actor, bypassing the tool registry, the selection gate, and the modal dialog entirely.
- `#3-headless-merge-rework` `IN-REVIEW` developer — Rewrote `performance.merge_actors` to run headless. The handler now collects each resolved actor's `UStaticMeshComponent`s with a non-null `GetStaticMesh()` directly (no editor-selection / `CanMergeFromSelection` dependency), computes an explicit output package (`outputPackage` param, default a unique `/Game/Merged/MERGED_<FirstActor>`), and calls `IMeshMergeUtilities::MergeComponentsToStaticMesh` (`MeshMergeModule.GetUtilities()`) — never `IMergeActorsTool::RunMergeFromSelection`, so `CreateModalSaveAssetDialog` is never reached. It then registers the produced assets, spawns an `AStaticMeshActor` for the merged `UStaticMesh` at the merge location, and destroys the sources when `replaceSourceActors` is true. The `toolName` doc is corrected; the legacy "MeshMerging" alias is mapped to "Merge" and other tool names are rejected. Files: `Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp` (handler + includes), `Source/PinWright/PinWright.Build.cs` (added `MeshMergeUtilities` dep). Regression test `PinWright.performance.merge_actors.HeadlessOutputPackageAndHonestToolName` (`Source/PinWright/Private/Tests/EditorOps/TestDebugHandlers.cpp`) pins both reworded defects from the registered param metadata: the `toolName` description must not claim the non-matching "defaults to MeshMerging" default (defect #1), and the handler must expose the `outputPackage` slot that proves the direct-utilities headless bypass (defect #2) — it would fail if the handler reverted to the modal-dialog tool path.
- `#4-reword-and-fix` `IN-REVIEW` developer — Reworded: three validity lenses (correctness, adversarial, board-history) confirmed root causes #1 (wrong documented `toolName`) and #3 (the always-modal `CreateModalSaveAssetDialog` in `RunMergeFromSelection`), but FALSIFIED the original "core bug" #2 against UE 5.7 engine source: `CanMergeFromSelection()` reads the same `GEditor->GetSelectedActors()` `USelection` the handler writes via `SelectActor(..., bNotify=true)`, so no `NoteSelectionChange` is needed and the selection-tweak step was dropped. Retitled/reframed around the two verified root causes and demoted the selection hypothesis to a retracted note. Fix implemented in `Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp`: (a) corrected the `toolName` param doc to "Merge" and added a case-insensitive "MeshMerging"->"Merge" alias before the tool-name match loop (rejecting other tool names with `MERGE_TOOL_UNAVAILABLE`); (b) replaced the modal-dialog `RunMergeFromSelection()` path with a direct headless merge — collect the static-mesh `UPrimitiveComponent`s from the resolved actors, compute the explicit output package, call `IMeshMergeUtilities::MergeComponentsToStaticMesh` (from the newly-linked `MeshMergeUtilities` module in `PinWright.Build.cs`) with that package, then spawn the merged `AStaticMeshActor` (into the source level) and (when `replaceSourceActors`) destroy the sources; the response reports the real `mergedPackageName`/`mergedAssetPath`. Regression test `FPerfMergeActorsToolNameAliasAndNoModalTest` added to `Source/PinWright/Private/Tests/EditorOps/TestDebugHandlers.cpp`: pins that the registered `toolName` doc advertises "Merge" (not the stale "MeshMerging" default) and that a live two-StaticMeshActor merge via the legacy alias is neither `MERGE_TOOL_UNAVAILABLE` nor `MERGE_NOT_POSSIBLE` and reports an explicit `mergedPackageName` (never the modal-path `defaultPackageName`), so reverting either fix fails the test.
- `#5-merge-resolve` `IN-REVIEW` developer — Two hosts independently fixed this ticket with the same engine approach (direct `IMeshMergeUtilities::MergeComponentsToStaticMesh`, headless, explicit package). Integrated both into one handler: kept the `toolName` validation + legacy-alias mapping and the same-level spawn from one side, and the `outputPackage` param + sanitize/`CreateUniqueAssetName` + path validation + asset registration + `UEditorActorSubsystem` deletion from the other. Both regression tests are retained (`FPerfMergeActorsToolNameAliasAndNoModalTest` — live-merge lens; `FPerfMergeActorsHeadlessParamsTest` — param-metadata lens) and both pass against the merged handler, which exposes `outputPackage`, documents `toolName` as "Merge", and reports `mergedPackageName` on success.
