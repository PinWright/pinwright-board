---
id: F-geometry-loft-true-multiprofile
title: "geometry.loft only sweeps a circle between the first and last profile — implement a true multi-profile loft"
status: OPEN
severity: Medium
category: feature
tags: [loft, geometry, silent-wrong-shape]
encounters: 1
lastSeen: 2026-07-02T05:22:47.0291434+03:00
---

# geometry.loft only sweeps a circle between the first and last profile — implement a true multi-profile loft

Even after the empty-mesh sweep-stub bug is fixed (see
`B-geometry-loft-silent-empty-mesh`, which makes the verb actually produce
geometry), `geometry.loft` is **not a true loft**. In the multi-profile branch of
`AdvancedMeshOpsHandler.cpp` it:

- uses only the **first and last** profile actors (`FirstProfile =
  ProfileMeshActors[0]`, `LastProfile = ProfileMeshActors.Last()`) — every
  intermediate profile is ignored;
- approximates the cross-section as a **circle** of the first profile's max XY
  bounding-box extent (`ProfileRadius = FMath::Max(ProfileExtent.X,
  ProfileExtent.Y)`) — the profile's real silhouette (a disc, a polygon, a
  non-circular outline) is discarded;
- sweeps that single circle in a **straight line** between the two actor
  locations.

So a request like the vase-style stack in the reporter's repro
(`Prof_Foot` r=16 → `Prof_Belly` r=52 → `Prof_Neck` r=24 → `Prof_Rim` r=34) still
reports `profilesUsed:4` but produces a plain circular tube of radius 16 from foot
to rim — a silent WRONG shape (correct triangle count, wrong geometry). The caller
has no signal that the intermediate profiles and the varying radii were dropped.

## What a true loft should do

Skin a surface through **all** the supplied cross-section profiles in order,
honoring each profile's actual outline and transform:

- sample each profile actor's cross-section (its boundary loop / silhouette, not a
  bounding-box circle);
- order profiles along the loft path by their actor positions;
- interpolate between consecutive profiles (matching vertex counts / resampling as
  needed) and stitch the skin, optionally capping the ends and applying smooth
  normals.

Geometry Script offers building blocks (e.g. per-profile polygon extraction plus a
lofted-skin/sweep with per-frame scale, or a mesh-from-planar-sections approach); a
straight radius sweep is a placeholder, not this.

**Workaround:** For axisymmetric turned shapes use `geometry.revolve` (sweep a
silhouette polyline around an axis), which produces a real varying-radius surface.
For a non-axisymmetric multi-profile skin there is currently no faithful verb.

**Fix:** Replace the first-and-last circle sweep with a real multi-section loft that
consumes every profile's silhouette and position. If some inputs cannot be honored
(e.g. mismatched topology), surface that in the response rather than silently
approximating.

## History
- `#1-split-from-empty-mesh-bug` `OPEN` reporter — Split out of `B-geometry-loft-silent-empty-mesh` during that ticket's reword: the empty-mesh fix makes loft emit geometry, but the multi-profile branch still ignores intermediate profiles and every profile's real silhouette (uses only first+last actors, approximates the section as a circle of the first profile's max XY extent, sweeps it straight between the two actor locations), so it produces a circular tube instead of the requested lofted surface — a silent wrong shape. Tracks implementing a faithful multi-profile loft.
