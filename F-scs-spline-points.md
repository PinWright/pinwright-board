---
id: F-scs-spline-points
title: "No RPC to author USplineComponent point positions/types on a Blueprint SCS template"
status: IN-REVIEW
severity: Medium
category: feature
tags: [spline, scs, blueprint, component-template, content-authoring]
---

# No RPC to author USplineComponent point positions/types on a Blueprint SCS template

Adding a `USplineComponent` to a Blueprint's SCS (class template) and setting its
**closed-loop** flag is already possible — `misc.create_spline_component`
(`Handlers/Utility/MiscHandler.cpp:360`) creates a `USplineComponent` SCS node
(`SCS->CreateNode(USplineComponent::StaticClass())` `:403`) and calls
`SplineComp->SetClosedLoop(bClosedLoop)` (`:412`); `blueprint.scs.add_component`
(`Handlers/Blueprint/SCSHandler.cpp:80`) also adds the component. What has **no**
RPC is authoring the spline's **point positions and point types** on that class
template.

Every spline point-authoring verb in `Handlers/Geometry/SplineHandler.cpp` is
**actor-scoped**: `spline.add_spline_point` (`:292`),
`set_spline_point_position` (`:414`), `set_spline_point_tangents` (`:472`),
`set_spline_point_rotation` (`:540`), `set_spline_point_scale` (`:598`), and
`set_spline_type` (`:656`) all take `actorName` and resolve a placed actor via
`FindActorByName` (`:67`) + `FindSplineComponent` (`:82`) — none can target a BP
SCS component template. The only other `blueprintPath`-taking spline verb,
`spline.create_spline_mesh_component` (`:730`), authors a `USplineMeshComponent`
(a static-mesh subclass, not a `USplineComponent`) and only sets mesh/axis/material.

On the SCS side, `blueprint.scs.set_property` (`SCSHandler.cpp:207`) can set a
scalar/array UPROPERTY, but it authors no `SplineCurves` point data and, crucially,
performs **no `UpdateSpline()` recompute** of the interp curves / reparam tables —
so even hand-writing the nested `FSplineCurves` struct through it would not rebuild
the spline. `UpdateSpline()` appears only in the actor-scoped `SplineHandler.cpp`
handlers.

**Workaround (removed by this fix):** `python.execute` —
`SubobjectDataSubsystem.k2_gather_subobject_data_for_blueprint(bp)`, match the
`<Name>_GEN_VARIABLE` template, then call `set_spline_points` /
`set_spline_point_type` on it. A source-dive fallback repeated ~32 times across
subagents in one gate-migration session.

**Fix:** Add `blueprint.scs.set_spline_points`
(`blueprintPath` + `componentName` + `points` array + optional `pointType` /
`closedLoop` / `save`) that resolves the `USplineComponent` SCS template, replaces
its points, sets the per-point type, optionally sets the closed-loop flag, and
calls `UpdateSpline()` to rebuild the reparam tables — the recompute
`blueprint.scs.set_property` cannot do. This is the SCS-namespace sibling of the
existing `blueprint.scs.add_component` / `misc.create_spline_component` create verbs.

## History
- `#1-initial-request` `OPEN` reporter — Migrating 32 race-gate Blueprints to add a closed `USplineComponent` opening to each gate's SCS. Per gate: `blueprint.scs.add_component {componentClass:"SplineComponent", componentName:"OpeningSpline"}` worked, `blueprint.scs.set_property {propertyName:"ComponentTags"}` worked, but authoring the spline point data FORCED a `python.execute` fallback (`k2_gather_subobject_data_for_blueprint` → match `OpeningSpline_GEN_VARIABLE` → `set_spline_points`/`set_closed_loop`/`set_spline_point_type` on the template) because no RPC exists. Verified in source: all `spline.*` point setters are actor-scoped (`SplineHandler.cpp:67,82,292,414,472,540,598,656`); `spline.create_spline_mesh_component` (`:730`) targets `USplineMeshComponent` only; `blueprint.scs.*` (`SCSHandler.cpp:80,207`) can add the component and set scalar props but never the `SplineCurves` point array. Severity Medium (not the proposed High): a working python workaround exists (soft blocker) and SCS-spline-template authoring is a below-average-reach path. Not a dup of `F-scs-dsl-sidecar` (read-side SCS dump DSL), `F-water-actor-authoring` (defers point authoring to the actor-scoped `spline.*`), or `E-spline-create-actorname-echoes-deduped` (actorName echo).
- `#2-reword` `OPEN` developer — Board-historian lens correction: the original premise "no RPC to set closed-loop on an SCS spline template" was inaccurate. `misc.create_spline_component` (`MiscHandler.cpp:360`) already creates the `USplineComponent` SCS node and sets closed-loop (`SetClosedLoop`, `:412`). Narrowed the real gap to spline **point positions + point types** (and the `UpdateSpline()` recompute), and pointed the fix at an SCS-namespace sibling verb rather than overloading the six actor-scoped `spline.*` setters. Severity Medium and category feature unchanged (the point-authoring gap that forced the ~32x python fallback is real).
- `#3-implement-set-spline-points` `IN-REVIEW` developer — Added `blueprint.scs.set_spline_points` in `Handlers/Blueprint/SCSHandler.cpp` (canonical `blueprint.scs.*` file, next to `add_component`/`set_property`): resolves the `USplineComponent` SCS template by variable name, `ClearSplinePoints` + `AddSplinePoint` (local space) for each `{x,y,z}`/`{location:{x,y,z}}` entry, `SetSplinePointType` per point, optional `SetClosedLoop` when `closedLoop` is present, then `UpdateSpline()` and `MarkBlueprintAsStructurallyModified` (+ optional `McpSafeAssetSave`). Response echoes `pointCount`/`splineLength`/`closedLoop`. Two uniquely-named file-local helpers (`SCSSplinePointTypeFromString`, `SCSReadSplinePointLocation`) to avoid Unity ODR collisions. Regression test `Tests/Blueprint/TestSCSSetSplinePoints.cpp` (`PinWright.blueprint.scs.set_spline_points.AuthorsSCSTemplate`) builds a transient BP with a spline SCS node, dispatches the verb with 3 points + `closedLoop`, and asserts on the actual component template that the 3 points landed, the closed-loop flag flipped, and `GetSplineLength() > 0` (the `UpdateSpline` recompute proof); reverting the handler makes the invoke fail to find the method and dropping `UpdateSpline` leaves length 0.
