---
id: F-blueprint-preview-construction
title: "No way to run a Blueprint's construction script and see what it builds: no preview verb, no rerun verb, and capture_asset_preview rejects Blueprints"
status: IN-REVIEW
severity: Medium
category: feature
tags: [blueprint, construction-script, preview, gap-analysis-2026-09-30]
encounters: 1
---

# No construction script preview

There is no rerun or preview verb.
- `RerunConstructionScripts` is only called at `EnvironmentHandler.cpp:272` (sky sphere).
- `blueprint.add_construction_script` (`BlueprintCompileHandler.cpp:63-119`) only creates the graph.
- `render.capture_asset_preview` rejects Blueprint subjects.
- `actor.set_blueprint_variables` is a raw reflection write with no notification (`ActorPropertyHandler.cpp:318+`).
- `property.set` does rerun construction on placed actors as an unreported side effect: see `B-property-set-wiki-construction-rerun`.

pinwright.com/compare rates the row partial. ue-mcp `RunConstructionScript` (`BlueprintHandlers.cpp:4017`) is a thin yes: a transient actor in the editor world, per-component name, class, relative transform and isRoot only, no variable overrides, no outliner hiding.

**Fix:** a new verb `blueprint.preview_construction` (in `BlueprintCompileHandler.cpp` or a new `BlueprintPreviewHandler.cpp`).
- Params: `path`, `world: preview|editor` (default `preview`, echoed; `editor` lets world-querying scripts see the level), `transform`, `variables{}`.
- Spawn deferred (`FActorSpawnParameters::bDeferConstruction` `World.h:476`, `bTemporaryEditorActor` `:486`, `bHideFromSceneOutliner` `:489`, `RF_Transient` via `ObjectFlags` `:515`).
- Apply `variables` with the same setter `actor.set_blueprint_variables` uses, then `FinishSpawning` (`Actor.h:3117`), read, and destroy (`EditorDestroyActor`).
- Model: `FBlueprintEditor::UpdatePreviewActor` (`BlueprintEditor.cpp:10569-10660`) with an `FPreviewScene` (`PreviewScene.cpp:38-39`). An existing transient-spawn pattern is at `LevelStructureHandler.cpp:2312-2325`.
- Response: `components[{name, class, creationMethod Native|SCS|UCS|Instance, attachParent, relativeTransform, worldTransform, properties (sparse diff vs template)}]`, `childActors[]`, `bounds`, `compileStatus`.
- Errors: refuse `BS_Error` with `BLUEPRINT_COMPILE_FAILED`; also `BLUEPRINT_NOT_FOUND`, `CLASS_NOT_INSTANTIABLE`, `ACTOR_SPAWN_FAILED`, `TYPE_MISMATCH` for a bad variable.
- No transaction: nothing persists. `MarkPackageDirty` early-outs on `RF_Transient` (`UObjectBaseUtility.cpp:236-245`).
- Defer `actor.rerun_construction` unless it is cheap to add.

**Acceptance:**
- A Blueprint whose construction script adds N components from an int variable returns N rows with `creationMethod: UCS`. With `variables{Count: M}` it returns M rows.
- The SCS rows match `blueprint.scs.get`.
- The map package stays clean, and no actor or preview world is left behind.
- A `BS_Error` Blueprint is refused.

Effort M. Risk: a construction script whose logic depends on the world behaves differently in the preview world; the `world` param discloses this.

## History
- `#1-gap-analysis` `OPEN` reporter — Filed from the 2026-09-30 competitor gap analysis (compare row "Construction script preview": PinWright partial, ue-mcp yes). Evidence and design above.
- `#2-preview-construction-verb` `IN-REVIEW` developer — Added `blueprint.preview_construction` in new `Source/PinWright/Private/Handlers/Blueprint/BlueprintPreviewHandler.cpp`. The verb resolves and loads the Blueprint. It refuses `CLASS_NOT_INSTANTIABLE` for a non-Actor, abstract or uncompiled class, and `BLUEPRINT_COMPILE_FAILED` when `UBlueprintGeneratedClass::GetGeneratedClassesHierarchy` reports an error Blueprint in the chain; that is the same gate under which `ExecuteConstruction` falls back to a billboard. It checks `variables` names before spawning (`PROPERTY_NOT_FOUND`). It spawns deferred with `bTemporaryEditorActor`, `bHideFromSceneOutliner` and `ObjectFlags = RF_Transient` (not transactional) into a private `FPreviewScene` (`world: preview`, default) or the editor world (`world: editor`). It applies `variables` with `ApplyJsonValueToProperty`, the setter `actor.set_blueprint_variables` uses (`TYPE_MISMATCH` on failure, and the instance is still destroyed). It then calls `FinishSpawning`, reads the instance, and destroys it with `EditorDestroyActor` and then the scene, suppressing `r.ForceGCOnPreviewSceneExit` the way `niagara.simulate` does. The response carries `world` (echoed), `compileStatus`, `userConstructionScriptEnabled` (`[Kismet] bTurnOffEditorConstructionScript`), `variablesApplied`, `spawnTransform`, `components[]`, `componentCount`, `childActors[]` and `bounds{valid,min,max}`. Component rows reuse `ActorDescribeBuilder::BuildComponentsJson` plus `worldTransform`. **Deviation from the design above:** `creationMethod` uses actor.describe's existing spelling (`Native` / `SimpleConstructionScript` / `UserConstructionScript` / `Instance`), not `SCS` / `UCS`, so both verbs share one vocabulary (rpc-design §21). The transform is passed as `location` / `rotation` / `scale`, the same as `actor.spawn`, instead of a `transform` object. Also added `blueprint.preview_construction` to SafePoint table B next to `niagara.simulate` (`Dispatch/SafePoint.cpp`), a `### blueprint.preview_construction` section in `docs/wiki-src/blueprint.md`, and a CHANGELOG entry. No new error codes. Tests in `Source/PinWright/Private/Tests/Blueprint/TestBlueprintPreviewConstruction.cpp`. The fixture is a transient Actor BP with one SCS SceneComponent and a UCS of the form `ForLoop(1..Count)` -> `AddComponentByClass(SceneComponent)`, where `Count` is an int with default 3. Test ids: `PinWright.blueprint.preview_construction.UcsRowsFollowTheCountVariable` (3 UCS rows by default, 5 with `variables{Count:5}`, the class default is unchanged afterwards, the SCS rows equal the SCS node variable names, and the world-context count is unchanged), `PinWright.blueprint.preview_construction.EditorWorldLeavesNoActorAndNoDirtyMap` (the map package stays clean and no live instance remains, after both success and a `TYPE_MISMATCH` refusal) and `PinWright.blueprint.preview_construction.RefusesErrorBlueprintAndUnknownVariable`. Not done: `actor.rerun_construction` (deferred, as the Fix allows) and Blueprint capture support, which was split to F-capture-asset-preview-no-blueprint-subject-kind.
