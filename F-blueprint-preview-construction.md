---
id: F-blueprint-preview-construction
title: "No way to run a Blueprint's construction script and see what it builds: no preview verb, no rerun verb, and capture_asset_preview rejects Blueprints"
status: OPEN
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
