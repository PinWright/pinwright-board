---
id: E-geometry-mesh-info-omits-bbox
title: "geometry.get_mesh_info omits the mesh bounding box, so any extent/size check after a topology op forces a separate actor.get_bounding_box round-trip — even though sibling geometry handlers already read GetMeshBoundingBox off the same Target.Mesh"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [geometry, get_mesh_info, bounding-box, extent, readback, round-trip, response-shape, mesh-info]
---

# geometry.get_mesh_info reports vertex/triangle counts and feature flags but no bounding box, so verifying a clip/extent always costs a second actor.get_bounding_box call

`geometry.get_mesh_info {actorName}` returns `vertexCount`, `triangleCount`,
`hasNormals`, `hasUVs`, `hasColors`, `hasPolygroups`
(MeshInfoHandler.cpp:101-107) — the topology head-count and the feature flags,
but **no spatial extent**. There is no `boundingBox` / `bounds` / `extent`
field in the response. So whenever a build needs to confirm *where the mesh
ended up* — that a boolean clip clamped the shape to a box, that a deform did
not blow the silhouette out, that a baked prop is the size the level expects —
`get_mesh_info` cannot answer, and the caller must fire a separate
`actor.get_bounding_box` to read the extent.

This is the read-verb analog of `E-geometry-deformer-echo-mesh-counts` (which
covers the *mutators* not echoing counts). That ticket's history #2 already
flags this same hole as "see also the bbox gap" — `get_mesh_info also returns
no bbox" — but no dedicated ticket existed for it. This is that ticket: the gap
is in the **read verb itself**, not in any mutator.

The bounding box is trivially available on the exact `Target.Mesh` the handler
already holds. Sibling geometry handlers in the same plugin already call it:
`MeshOpsHandler.cpp` and `AdvancedMeshOpsHandler.cpp` both read
`UGeometryScriptLibrary_MeshQueryFunctions::GetMeshBoundingBox(Target.Mesh)` →
`FBox` (origin/extent/min/max). `get_mesh_info` runs `GetVertexCount` and
`GetTriangleCount` on that same `Target.Mesh` (MeshInfoHandler.cpp:93-94); one
more `GetMeshBoundingBox` call on the same handle would fold the extent into
the info response.

The distinction from `actor.get_bounding_box` matters: `actor.get_bounding_box`
reports the **actor's world-space component bounds** (world transform applied),
whereas `get_mesh_info` describes the **dynamic mesh's local geometry**. For a
mesh authored at the origin with identity transform (the common modeling-tools
case) they coincide, which is why the workaround works — but a caller who has
moved/rotated/scaled the actor and wants the *local* mesh extent has no read
verb that returns it at all. Surfacing the local `GetMeshBoundingBox` on
`get_mesh_info` both removes the round-trip for the common case and exposes the
local-space extent that `actor.get_bounding_box` cannot.

## What it should do

`geometry.get_mesh_info` should add a `boundingBox` object (local mesh space)
sourced from `UGeometryScriptLibrary_MeshQueryFunctions::GetMeshBoundingBox(Target.Mesh)`
— at minimum `{min, max, origin, extent}`, matching the field shape the
asset-dump builders already emit for static/skeletal meshes
(`bounds.{origin,extent,sphereRadius}`). The handler's own registration string
already promises "vertex count, triangle count, **etc.**" — the extent is the
most-requested "etc." after counts, and it is one GeometryScript call away on a
mesh the handler is already holding.

**Workaround:** call `actor.get_bounding_box {actorName}` after `get_mesh_info`
to read the extent (which is exactly what this task did to verify the
box-clip). This only matches the mesh-local extent when the actor's transform
is identity.

## Friction evidence (this task — geometry.boolean_intersection "cushion gem" build, 10 calls, outcome clean, friction:"none")

The story's success criterion (step 6) was to confirm "the shape was clipped to
the box's extents" after the boolean intersection. `get_mesh_info` confirmed the
topology change (BEFORE v3176/t6348 → AFTER v1733/t3462) but **could not** report
the extent, so the agent had to add a separate `actor.get_bounding_box
{GemSphere}` → extent `[45,45,45]` to prove the clip clamped to the 90³ box's
half-extent. The friction note names this exactly: *"the only nuance was
get_mesh_info returns no bounding box, so I used actor.get_bounding_box (a
natural, documented method) to verify the box clip extent."* Every call
first-try clean, judge filed nothing — pure PROCESS overhead: one extra
round-trip per "did the op land where expected?" check, which a `boundingBox`
field on the info response would eliminate. This is the verify-extent shape, and
it recurs anywhere a build asserts a mesh's size/clip rather than just its
head-count.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.boolean_intersection` "cushion gem" prop task (10 calls, outcome clean, friction:"none", judge filed nothing). PROCESS finding: `geometry.get_mesh_info` returns `vertexCount`/`triangleCount`/`hasNormals`/`hasUVs`/`hasColors`/`hasPolygroups` (MeshInfoHandler.cpp:101-107) but **no bounding box / extent**, so the story's "confirm the shape was clipped to the box's extents" step forced a separate `actor.get_bounding_box {GemSphere}` → `[45,45,45]` after `get_mesh_info` (which confirmed only the v/t change 3176/6348 → 1733/3462). The local mesh bbox is one `UGeometryScriptLibrary_MeshQueryFunctions::GetMeshBoundingBox(Target.Mesh)` call away on the same handle the info verb already holds — sibling handlers `MeshOpsHandler.cpp`/`AdvancedMeshOpsHandler.cpp` already read it. This is the dedicated ticket for the "bbox gap" that `E-geometry-deformer-echo-mesh-counts #2` named as "see also" but never filed; distinct from that ticket (mutators omitting count echoes) and from `E-geometry-create-name-vs-actorname` (param-slot drift). Fix: add a `boundingBox {min,max,origin,extent}` field to the `get_mesh_info` response. Workaround: `actor.get_bounding_box` after `get_mesh_info` (matches only at identity actor transform).
- `#3-fix` `IN-REVIEW` developer — Fixed. `geometry.get_mesh_info` now folds the dynamic mesh's local-space bounding box into its response as `boundingBox {min, max, origin, extent}` (each a `{x,y,z}`), sourced from `UGeometryScriptLibrary_MeshQueryFunctions::GetMeshBoundingBox(Target.Mesh)` — the same `Target.Mesh` handle the existing vertex/triangle counts already read (`MeshInfoHandler.cpp`, after the existing `Result->Set*` block; same FBox sibling handlers spherify/cylindrify/AdvancedMeshOps already consume). So the "did the op land at the expected size/extent?" check no longer forces a separate `actor.get_bounding_box` round-trip, and unlike that verb this surfaces the mesh-LOCAL extent (the two diverge once the actor carries a non-identity transform). Also: updated the handler's registration summary to name the boundingBox instead of a vague "etc.", and added a `### geometry.get_mesh_info` H3 overlay section to `docs/wiki-src/geometry.md` documenting the new field + the local-vs-world distinction. No aspect-version bump (this is a live RPC response, not a cached asset-dump aspect). Files: `Source/PinWright/Private/Handlers/Geometry/MeshInfoHandler.cpp`, `docs/wiki-src/geometry.md`. Regression test: `PinWright.geometry.get_mesh_info.ReportsBoundingBox` in `Source/PinWright/Private/Tests/World/TestGeometryHandlers.cpp` — spawns a non-cubic box (80x200x120) via the production `geometry.create_box`, calls the production `geometry.get_mesh_info`, and asserts the response carries a `boundingBox` whose `min/max/origin/extent` sub-objects are present, internally consistent (extent == half the min->max span, origin == midpoint), centered at the local origin, and whose sorted half-extents equal the box's half-dimensions {40,60,100}. Reverting the fix drops the `boundingBox` field and fails every assertion.
- `#2-additional-archbeam-bend-deform` `OPEN` reporter — Additional evidence (seed `geometry.bend`, "ArchBeam_DM" stylized-archway build, 11 calls, outcome clean, all RPCs first-try clean). The story called the bbox gap out **twice over**: step 2 asked to "read back its mesh info (vertex count, triangle count, **bounds**)" and step 6 asked to "read the mesh info again … confirm the bends actually changed the **geometry's extent** (the bent/tapered/twisted result should have a noticeably different bounding box than the straight cylinder)". `geometry.get_mesh_info` returned only counts/flags (baseline 626v/1248t), so **both** bounds reads fell through to a separate `actor.get_bounding_box {ArchBeam_DM}` — baseline extent 30/30/200 vs post-deform extent 51.9/97.8/191.4 (origin shifted to y=47.2 as the beam curled over). The attempt agent's friction note names it verbatim: *"geometry.get_mesh_info returns ONLY vertex/triangle counts and has* flags — NO bounding box, despite the task/user asking to 'read back bounds' from it … bounds belong in get_mesh_info's output"* (the agent independently re-read MeshInfoHandler.cpp to confirm the omission). This is the strongest verify-extent recurrence yet: the user phrased the success criterion *as a bounding-box comparison*, and a `boundingBox` field on the info response would have folded both `actor.get_bounding_box` round-trips back into the `get_mesh_info` calls already in the chain. Same fix/workaround as #1. Note: this build was the seed for `geometry.bend`, which itself behaved correctly (deform landed, bounds changed as expected) — the friction is purely the read-verb's omitted bbox, hence culprit `geometry.get_mesh_info`.
