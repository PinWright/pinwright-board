---
id: B-geometry-sweep-spline-frames-world-space
title: "geometry.sweep / extrude_along_spline feed WORLD-space spline frames into the target's LOCAL mesh"
status: IN-REVIEW
severity: Medium
category: bug
tags: [geometry, sweep, spline, coordinate-space, silent-wrong-shape]
encounters: 1
lastSeen: 2026-09-30T05:30:00+03:00
---

# geometry.sweep / extrude_along_spline feed WORLD-space spline frames into the target's LOCAL mesh

`AdvancedMeshOpsHandler.cpp`'s `SampleSplineFrames` samples the spline with
`ESplineCoordinateSpace::World`, and both `geometry.sweep` and
`geometry.extrude_along_spline` hand those frames straight to the op, which appends vertices
to the target actor's `UDynamicMesh` - a buffer in the target actor's LOCAL space. Nothing
applies the inverse of the target actor's transform.

So whenever the target actor is not at the identity transform, the swept tube lands offset
by the target's location (and rotated/scaled by its rotation/scale) away from the spline it
was asked to follow, while the response reports success. Found in passing while fixing
`F-geometry-loft-true-multiprofile` (which converts its profile locations into target-local
space in the same file).

**Repro (reasoned, not yet run live):** create a procedural-mesh target at (1000,0,0), a spline
actor at the origin, `geometry.sweep {actorName, splineActorName}`; the tube's world bounds
are centred near X=1000 instead of on the spline.

**Fix sketch:** transform each sampled frame by
`Target.Actor->GetActorTransform().Inverse()` (or sample in local space relative to the target)
before handing it to the op; regression test with a target placed off the origin.

## History
- `#1-found-during-loft-fix` `OPEN` developer — Filed while implementing `F-geometry-loft-true-multiprofile`: sweep and extrude_along_spline sample splines in world space and append to a local-space mesh with no conversion.
- `#2-target-local-frames` `IN-REVIEW` developer — `SampleSplineFrames` (the one sampler both verbs call) now takes the target component's transform and returns each frame through `InverseTransformRotation` / `InverseTransformPosition`, so the frames reach the op in the mesh's local space; `geometry.sweep` passes `Target.Component->GetComponentTransform()`, `geometry.extrude_along_spline` passes `DMC->GetComponentTransform()`. Files: `Source/PinWrightGeometry/Private/Handlers/Geometry/AdvancedMeshOpsHandler.cpp`, new `Source/PinWrightGeometry/Private/Tests/Geometry/TestGeometrySweepSplineTargetLocalFrames.cpp`, `docs/wiki-src/geometry.md` (sweep + extrude_along_spline bullets), `CHANGELOG.md`. Tests: `PinWright.geometry.sweep.SplineFramesFollowSplineOffOrigin`, `PinWright.geometry.extrude_along_spline.SplineFramesFollowSplineOffOrigin` — spline actor at (-400,300,50) yaw 30, target at (1000,-500,200) pitch 20 yaw 90 scale 1.5; every mesh vertex taken to world space must lie within 1.05 x 75 + 2 uu of the spline curve (fails by >1000 uu with the fix reverted).
