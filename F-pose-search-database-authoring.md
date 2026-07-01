---
id: F-pose-search-database-authoring
title: "Zero coverage for Pose Search / Motion Matching pipeline"
status: DONE
severity: Medium
category: feature
tags: [animation, pose-search, motion-matching, asset-authoring, plugin-gated]
---

# Zero coverage for Pose Search / Motion Matching pipeline

`Plugins/Animation/PoseSearch` ships the asset types that drive UE's Motion
Matching workflow — `UPoseSearchSchema`, `UPoseSearchDatabase`, the
`UPoseSearchFeatureChannel*` family, plus the AnimGraph nodes
`UAnimGraphNode_PoseSearchSearchContinuously` /
`UAnimGraphNode_MotionMatching`. The MCP plugin currently has **zero**
references to any of this surface: a grep for `pose_search`, `PoseSearch`,
`motion_match`, `MotionMatching` across
`docs/rpc-method-reference.generated.md` and
`Source/EditorAutomationRpcGateway/Private/Handlers/` returns nothing.

Effect: agents authoring locomotion / motion-matching pipelines cannot
create or populate the schema and database assets that Motion Matching
needs, nor wire up the runtime AnimGraph nodes that consume them. They
have to fall back to manual editor workflows or to writing/loading
.uasset bytes directly.

**Proposal:** Add a `pose_search.*` namespace with the small set of
authoring RPCs that cover the end-to-end pipeline:

1. `pose_search.create_schema(assetPath, skeleton, channels[])` — creates
   a new `UPoseSearchSchema` at `assetPath`, sets the target skeleton,
   appends each entry in `channels[]` as a typed
   `UPoseSearchFeatureChannel*` (Position / Velocity / Trajectory /
   Heading / Phase) with bone reference + sample-time parameters.
2. `pose_search.create_database(assetPath, schema, animations[])` —
   creates a new `UPoseSearchDatabase`, binds the schema, and appends
   each entry in `animations[]` as a database animation entry
   (`FPoseSearchDatabaseAnimationAssetBase` subclass — Sequence /
   BlendSpace / AnimComposite) with a sampling range.
3. `pose_search.add_database_animation(assetPath, sequencePath,
   samplingRange)` — incremental append to an existing database, for
   callers that build the database iteratively.

For the AnimGraph side, fold into the generic
`animation.authoring.add_graph_node` proposed in
[`F-anim-add-graph-node-generic`](F-anim-add-graph-node-generic.md) — the
`UAnimGraphNode_PoseSearchSearchContinuously` /
`UAnimGraphNode_MotionMatching` classes are stock `UAnimGraphNode_*`
descendants and should resolve through the generic class catalog once
that ticket lands. Only add a `pose_search.add_motion_matching_node`
helper if profiling shows the property-dictionary form is too verbose
for the typical setup.

**Plugin gating:** The PoseSearch plugin is **not** enabled in every UE
project — `.uproject` files routinely turn it off. Handlers must gate on
module presence:

```cpp
if (!FModuleManager::Get().IsModuleLoaded(TEXT("PoseSearch")))
{
    Ctx.SendError(TEXT("PLUGIN_DISABLED"),
        TEXT("PoseSearch module is not loaded — enable the plugin in .uproject"));
    return true;
}
```

Pattern matches how the plugin already conditionally wires MetaSound,
StateTree, SmartObjects, MassEntity in `Build.cs` via
`TryAddConditionalModule()`. Build-side, the PoseSearch dependency
should be added through the same conditional mechanism so the handler
TU only compiles its PoseSearch-typed code when the engine version
ships the module — defensive against the plugin being moved or renamed
between 5.4 and 5.7.

**Use cases unlocked:**

1. Authoring greenfield Motion Matching setups from spec (schema +
   database + AnimBP wiring) without manual editor work.
2. Bulk-rebuilding databases from a generated animation list (e.g.
   regenerating a locomotion database after a re-export sweep).
3. Programmatic schema tweaks during iteration (adding a new bone
   channel, changing trajectory sample times) without touching the
   .uasset directly.

**Cross-ref:** Pairs with
[`F-anim-add-graph-node-generic`](F-anim-add-graph-node-generic.md) for
the AnimGraph node side. Independent of
[`F-search-api-anim-graph-nodes`](F-search-api-anim-graph-nodes.md)
because schema/database are content-browser assets, not graph nodes.

## History
- `#1-zero-coverage-confirmed` `OPEN` reporter — Grep over `docs/rpc-method-reference.generated.md` and `Source/EditorAutomationRpcGateway/Private/Handlers/` for `pose_search`, `PoseSearch`, `motion_match`, `MotionMatching` returned zero hits. `Plugins/Animation/PoseSearch` ships `UPoseSearchSchema` / `UPoseSearchDatabase` / `UPoseSearchFeatureChannel*` plus `UAnimGraphNode_PoseSearchSearchContinuously` / `UAnimGraphNode_MotionMatching`. Proposes `pose_search.create_schema`, `pose_search.create_database`, `pose_search.add_database_animation`, with the AnimGraph node side folding into the generic `animation.authoring.add_graph_node` from `F-anim-add-graph-node-generic`. Handlers gate on `FModuleManager::IsModuleLoaded("PoseSearch")` and return `PLUGIN_DISABLED` when the plugin is off in `.uproject` — same conditional-module pattern the plugin already uses for MetaSound / StateTree / SmartObjects.
- `#2-add-pose-search-authoring` `IN-REVIEW` developer — Added gated `pose_search.create_schema`, `pose_search.create_database`, and `pose_search.add_database_animation` handlers, documented the Pose Search pipeline, and added `FPoseSearchSchemaDatabaseAuthoringPipelineTest` coverage.
- `#3-review-iteration-1-fixes` `IN-REVIEW` developer — Tightened PoseSearch build-rule discovery, reused shared path/load/save helpers in the handler, and rejected database animations whose skeleton is incompatible with the bound schema.
- `#4-verify-mismatch-rejection` `DONE` tester — Verified: `pose_search.create_schema` created `/Game/EditorAutomationRpcGatewayTests/PSSchema_Verify_FPoseSearchDatabaseAuthoring_Mannequin` with `skeletonCount: 1`, `channelCount: 1`, `saved: false`; `pose_search.create_database` with that schema and mismatched `/App/Meshes/Truck/SK_Truck_Anim.SK_Truck_Anim` returned `SKELETON_MISMATCH` before database creation.
