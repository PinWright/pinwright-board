---
id: B-get-splines-info-ignores-spline-mesh
title: "spline.get_splines_info cannot see SplineMeshComponents created by spline.create_spline_mesh_actor"
status: IN-REVIEW
severity: High
category: bug
tags: [spline, spline-mesh, readback, get_splines_info, class-mismatch]
---

# spline.get_splines_info cannot see SplineMeshComponents created by spline.create_spline_mesh_actor

`spline.create_spline_mesh_actor` spawns an actor whose root is a
`USplineMeshComponent` (see `SplineHandler.cpp` line 1082:
`NewObject<USplineMeshComponent>(NewActor, *ComponentName)`). But the
companion readback `spline.get_splines_info` only iterates
`USplineComponent` — its `FindSplineComponent` helper calls
`Actor->GetComponents<USplineComponent>(...)` (lines 81-103), and the
no-arg list path does the same (lines 1338-1339). Since
`USplineMeshComponent` derives from `UStaticMeshComponent`, **not** from
`USplineComponent`, the readback is architecturally unable to report any
actor produced by `create_spline_mesh_actor`:

- Named lookup (`actorName` provided) returns the misleading error
  `[NO_SPLINE] No spline component found on actor`, even though the actor
  verifiably owns a `SplineMeshComponent`.
- No-arg list omits the spline-mesh actor entirely (it never appears in
  `splines[]` / `totalSplineActors`).

This is a coverage bug in `get_splines_info`, not a missing capability:
the create/configure/edit RPCs for spline-mesh actors all work and
round-trip fine (`configure_spline_mesh_axis`, `set_spline_mesh_asset`,
`set_spline_mesh_material` each find the component via
`GetComponents<USplineMeshComponent>` and return `existsAfter:true`), yet
the one prescribed readback in the same `spline` namespace cannot confirm
the result. There is no sibling read RPC that reports a SplineMeshComponent's
forward axis / mesh / state either — `actor.get_components` only confirms the
component class exists, not its spline-mesh configuration. The net effect is
a user can build and configure a spline-mesh handrail but has no in-namespace
way to read it back; the readback they are pointed to actively lies with
`NO_SPLINE`.

**What it should do:** `get_splines_info` should also recognize
`USplineMeshComponent` (and/or a dedicated `get_spline_mesh_info` should
exist) — for a named spline-mesh actor, report the component name, forward
axis, and start/end tangents rather than `[NO_SPLINE]`; for the no-arg
list, include spline-mesh actors so they aren't silently dropped.

**Workaround:** Verify spline-mesh actors with `actor.get_components`
(reports `class: /Script/Engine.SplineMeshComponent`) — but this does not
expose the forward axis or any spline-mesh state.

**Fix:** In `SplineHandler.cpp`, broaden the readback. Either teach
`get_splines_info` to fall back to `GetComponents<USplineMeshComponent>`
when no `USplineComponent` is found (emitting forward-axis / start-end
fields for those), or add a `spline.get_spline_mesh_info` RPC mirroring the
configure/set RPCs' `GetComponents<USplineMeshComponent>` + name-match
lookup.

## Verbatim repro (live, replay-confirmed)

1. `spline.create_spline_mesh_actor` args `{"actorName":"DemoRoom_Handrail","componentName":"RailMesh","meshPath":"/Engine/BasicShapes/Cube.Cube","forwardAxis":"X","location":{"x":400,"y":0,"z":120}}`
   → success: `{"actorName":"DemoRoom_Handrail","componentName":"RailMesh","existsAfter":true,"componentClass":"SplineMeshComponent",...}`
2. `spline.get_splines_info` args `{"actorName":"DemoRoom_Handrail"}`
   → error: `[NO_SPLINE] No spline component found on actor`
3. `spline.get_splines_info` args `{}` (list all)
   → handrail absent; only `TestSplineActor` entries, `totalSplineActors:7`.
4. `actor.get_components` args `{"actorName":"DemoRoom_Handrail"}`
   → confirms `{"name":"RailMesh","class":"/Script/Engine.SplineMeshComponent",...}`, `count:1` — the component is genuinely there.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed against live editor (UE 5.7). `create_spline_mesh_actor` makes a `USplineMeshComponent`; `get_splines_info` only scans `USplineComponent` (a non-parent class via `UStaticMeshComponent`), so the named readback returns a misleading `[NO_SPLINE]` and the no-arg list omits the actor, while `actor.get_components` proves the SplineMeshComponent exists. Root cause in `SplineHandler.cpp`: `FindSplineComponent` (lines 81-103) + no-arg loop (lines 1338-1339) both use `GetComponents<USplineComponent>`. Culprit method: `spline.get_splines_info` (seed was `spline.create_spline_mesh_actor`).
- `#2-fix` `IN-REVIEW` developer — Broadened `spline.get_splines_info` to recognize `USplineMeshComponent` in `SplineHandler.cpp`. Added namespace helpers `FindSplineMeshComponent`, `SplineMeshAxisToString`, and `FillSplineMeshFields` (forward axis + start/end position + start/end tangent, mirroring the configure/set RPCs' `GetComponents<USplineMeshComponent>` lookup). Named-lookup path now falls back to the spline-mesh component instead of returning `[NO_SPLINE]` (emits `isSplineMesh:true`, `componentName`, `forwardAxis`, `startPosition/startTangent/endPosition/endTangent`); the no-arg list path now includes spline-mesh-only actors instead of silently dropping them. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Geometry/SplineHandler.cpp`. Regression test: `EditorAutomationRpcGateway.spline.get_splines_info.SeesSplineMesh` in `Source/EditorAutomationRpcGateway/Private/Tests/World/TestGeometryHandlers.cpp` — spawns a spline-mesh actor via `create_spline_mesh_actor`, asserts the named readback succeeds (not `NO_SPLINE`) and reports `forwardAxis:Y`, and that the no-arg list includes the actor; fails if the fix is reverted. Not compiled/run yet (later phase).
