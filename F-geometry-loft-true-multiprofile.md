---
id: F-geometry-loft-true-multiprofile
title: "geometry.loft uses only the first+last profile (circle tube) — loft through every profile's radius+position"
status: IN-REVIEW
severity: Medium
category: feature
tags: [loft, geometry, silent-wrong-shape]
encounters: 1
lastSeen: 2026-07-02T05:22:47.0291434+03:00
---

# geometry.loft uses only the first+last profile (circle tube) — loft through every profile's radius+position

Even after the empty-mesh sweep-stub bug is fixed (see
`B-geometry-loft-silent-empty-mesh`, whose fix — the wired `AppendSweepPolygonCompat`
call — is already in plugin HEAD, so loft emits geometry), the multi-profile branch of
`geometry.loft` (`AdvancedMeshOpsHandler.cpp`) is **not a true loft**. It:

- reads only the **first and last** profile actors (`FirstProfile =
  ProfileMeshActors[0]`, `LastProfile = ProfileMeshActors.Last()`) — every
  intermediate profile is ignored;
- approximates the cross-section as a **circle** of the *first* profile's max XY
  bounding-box extent (`ProfileRadius = FMath::Max(ProfileExtent.X,
  ProfileExtent.Y)`) — every other profile's radius is discarded;
- sweeps that single circle in a **straight line** between the two actor locations,
  ignoring intermediate positions;
- still reports `profilesUsed = ProfileMeshActors.Num()`.

So the vase-style stack in the reporter's repro (`Prof_Foot` r=16 → `Prof_Belly`
r=52 → `Prof_Neck` r=24 → `Prof_Rim` r=34) reports `profilesUsed:4` but produces a
plain circular tube of radius 16 from foot to rim — a silent WRONG shape (correct
triangle count, wrong geometry). The caller gets no signal that the intermediate
profiles and the varying radii were dropped.

## Scope (reworded)

The worth-doing kernel is a **true multi-section loft over every profile's radius and
position**, plus honest reporting — NOT a full silhouette-faithful skinning engine. The
observed defect (dropped intermediate profiles + varying radii → wrong shape) is
axisymmetric in the only demonstrated case. Faithful *non-circular / non-axisymmetric*
silhouette extraction (boundary-loop sampling, cross-profile vertex resampling/matching,
stitching arbitrary outlines) is a large, fragile feature with no demonstrated demand
and is **explicitly out of scope here** — axisymmetric turned shapes are already served
by `geometry.revolve`. Reopen a follow-up only if a concrete non-axisymmetric profile
need is shown.

## Fix

Replace the first-and-last circle sweep with a real multi-section loft that consumes
**every** profile actor:

- read each profile's cross-section radius (max XY bbox extent) **and** its actor
  location;
- skin a ring through every profile in order (stitched consecutive rings, capped ends),
  so intermediate profiles and varying radii produce the correct varying-radius surface
  (the Foot→Belly→Neck→Rim vase) instead of a uniform first-radius tube;
- approximate each cross-section as a circle of that max-XY extent — a *documented*
  approximation — and **surface it in the response**: report `crossSection:"circle"`
  and the per-profile `profileRadii` actually consumed (plus `unhonoredProfiles` for any
  profile whose mesh could not be read), so the caller is never handed a silent wrong
  shape. `profilesUsed` now reflects the sections actually consumed.

**Workaround (until released):** for axisymmetric turned shapes use `geometry.revolve`
(sweep a silhouette polyline around an axis), which produces a real varying-radius
surface. For a non-axisymmetric multi-profile skin there is currently no faithful verb.

## History
- `#1-split-from-empty-mesh-bug` `OPEN` reporter — Split out of `B-geometry-loft-silent-empty-mesh` during that ticket's reword: the empty-mesh fix makes loft emit geometry, but the multi-profile branch still ignores intermediate profiles and every profile's real silhouette (uses only first+last actors, approximates the section as a circle of the first profile's max XY extent, sweeps it straight between the two actor locations), so it produces a circular tube instead of the requested lofted surface — a silent wrong shape. Tracks implementing a faithful multi-profile loft.
- `#2-reword-implement-circle-multisection` `IN-REVIEW` developer — Reworded scope: descoped full silhouette-faithful / non-axisymmetric skinning (gold-plating with no demonstrated demand — axisymmetric is served by `geometry.revolve`) down to a true multi-section loft over every profile's radius+position with honest reporting. Implemented in `Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/AdvancedMeshOpsHandler.cpp`: the multi-profile branch now reads EVERY profile's max-XY radius and actor location, stitches a ring through every profile (twist-free basis perpendicular to the first→last axis, consecutive-ring quads, capped ends, built directly on `FDynamicMesh3`), and the response now carries `crossSection:"circle"`, per-profile `profileRadii`, and `unhonoredProfiles`; `profilesUsed` reflects sections actually consumed. Regression test `PinWright.geometry.LoftMultiProfileHonorsAllProfiles` (`Tests/Geometry/TestGeometryLoftMultiProfileHonorsAllProfiles.cpp`) lofts a Foot16→Belly52→Neck24→Rim34 stack and asserts the widest INTERMEDIATE profile (Belly r=52) drives the surface width (`get_mesh_info` local bbox max-XY extent > 40, vs ~16 under the old first+last code) plus the new `profileRadii`/`crossSection` fields — all three assertions fail if the fix is reverted.
