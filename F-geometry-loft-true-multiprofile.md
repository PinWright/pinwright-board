---
id: F-geometry-loft-true-multiprofile
title: "geometry.loft uses only the first+last profile (circle tube) — loft through every profile's radius+position"
status: IN-REVIEW
severity: High
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
- `#3-attempt-failed` `OPEN` developer — Auto-fix attempt reached IMPL-UNVERIFIED; reverted and NOT pushed (build/tests not green).
- `#4-re-rated` `OPEN` triager — Severity Medium -> High. Re-checked at plugin HEAD: `AdvancedMeshOpsHandler.cpp` still samples extent from the first profile only and location from the first and last, while `profilesUsed` reports every profile passed, so a multi-profile loft returns a wrong shape under a response claiming all profiles were used — silent wrong data on the verb's normal multi-profile path, which is the rubric's High; not a rare edge path, so no bump down.
- `#5-implement-every-profile-ring` `IN-REVIEW` developer — Implemented in the `PinWrightGeometry` module. `GeometryOps::Loft` (`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps_Advanced.cpp`) now lofts through every profile that has a mesh: a unit-circle section swept through one path frame per profile location, each frame scaled to that profile's max-XY extent, with `subdivisions` linearly interpolated steps per span (so a two-profile loft keeps its old ring count); every ring shares one orientation perpendicular to the first->last axis. Two further defects in the old branch went with it: its frames were `FindBetweenNormals(Up, Direction)`, which puts the section plane parallel to a vertical path (a flat ribbon), and its sweep transform translated the whole loft by the first profile's location a second time. `FLoftOutputs` gains `ProfileRadii`; `ProfilesUsed` counts rings actually built. The handler (`AdvancedMeshOpsHandler.cpp`) now fills every profile's bounding-box extent, converts profile locations into the target actor's local space, and the response adds `crossSection: "circle"`, `profileRadii` and `unhonoredProfiles` (unresolved or mesh-less profile names; those are skipped). Tests: new `PinWright.Geometry.Ops.Advanced.LoftSkinsEveryProfileRadius` (Foot16/Belly52/Neck24/Rim34 at X=500, asserts every vertex on each profile's plane sits at that profile's radius, a full ring per plane, `profilesUsed` 4 and `profileRadii`, plus a mesh-less middle profile is skipped with `profilesUsed` 3) and extended `PinWright.geometry.LoftSweepEmitGeometryNotSilentEmpty` Phase A (`crossSection`, `profileRadii`, an unresolvable name reported in `unhonoredProfiles`). Wiki: `docs/wiki-src/geometry.md` `### geometry.loft`, loft sentences in `model.md` / `model.authoring.md`, `docs/defect-backlog.md` D-05.
