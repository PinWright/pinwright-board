---
id: F-geometry-skeletal-mesh-roundtrip-verbs
title: "No verb creates a USkeletalMesh — add the Geometry Script skeletal round-trip (copy from / copy to / create asset)"
status: IN-REVIEW
severity: High
category: feature
tags: [skeleton, skeletal-mesh, geometry-script, asset-creation, capability-gap, dead-end]
---

# No verb creates a USkeletalMesh

Nothing in any namespace brings a `USkeletalMesh` into existence, and nothing writes skeletal mesh
geometry (vertices/triangles) at all. `NewObject<USkeletalMesh>` appears **only** under
`Private/Tests/` — never in a handler. At plugin HEAD
`9da255f6d0ef5017cfddee03cd9458266c99d764` (`Plugins/PinWright/`), the complete set of hits is:

```
Source/PinWright/Private/Tests/Assets/TestSkeletalMeshDescribeHandler.cpp:25
Source/PinWright/Private/Tests/Assets/TestSkeletalMeshDumpBuilder.cpp:51
Source/PinWright/Private/Tests/Assets/TestSkeletalMeshDumpBuilder.cpp:124
Source/PinWright/Private/Tests/Gameplay/TestAnimationHandlers.cpp:2573
Source/PinWright/Private/Tests/Gameplay/TestAnimationHandlers.cpp:2830
Source/PinWright/Private/Tests/Gameplay/TestAnimationHandlers.cpp:4172
Source/PinWright/Private/Tests/Gameplay/TestSkinWeightAudit.cpp:109
```

`SkeletalMeshFactory` and `CreateNewSkeletalMeshAssetFromMesh` return zero hits anywhere in
`Source/`.

## The dead end that looks like a working path

`skeleton.create_skeleton` **does** exist (`Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp:486`),
and the `skeleton.*` namespace has 41 registered verbs that edit an existing mesh or skeleton
thoroughly — bones, sockets, virtual bones, weights, morph targets, cloth, physics bodies and
constraints. So the namespace reads as a complete authoring surface. An agent creates a `USkeleton`,
adds bones, and then discovers there is no verb that gives it a mesh, and no verb that authors
geometry for one. The gap is only visible after the work has already been committed to a path that
cannot finish.

The practical consequence, lived in this host project: **all skeletal-mesh authoring left the typed
API entirely and ran as raw Python through `python.execute`.** Nothing about that work is
inspectable, retryable, or error-coded by the plugin.

## The fix is small, and the path is already proven in this repo

Three verbs, wrapping the Geometry Script entry points this project already used successfully:

1. `geometry.copy_mesh_from_skeletal_mesh` — `UGeometryScriptLibrary_StaticMeshFunctions::CopyMeshFromSkeletalMesh`,
   mirroring the existing `geometry.create_from_static_mesh` (`MeshAssetIOHandler.cpp`).
2. `geometry.copy_mesh_to_skeletal_mesh` — the write-back.
3. `geometry.create_skeletal_mesh_from_dynamic_mesh` — `UGeometryScriptLibrary_CreateNewAssetFunctions::CreateNewSkeletalMeshAssetFromMesh`.

Feasibility, checked rather than assumed:

- **No `Build.cs` change.** `GeometryScriptingCore` *and* `GeometryScriptingEditor` are both already
  linked at `Source/PinWrightGeometry/PinWrightGeometry.Build.cs:54`.
  `CopyMeshFromSkeletalMesh`/`CopyMeshToSkeletalMesh` live in
  `GeometryScriptingCore/Public/GeometryScript/MeshAssetFunctions.h`;
  `CreateNewSkeletalMeshAssetFromMesh` in
  `GeometryScriptingEditor/Public/GeometryScript/CreateNewAssetUtilityFunctions.h`. Nothing new to
  link.
- **No version gating.** All three exist on UE 5.3 through 5.8. Verified `CopyMeshFromSkeletalMesh`
  + `CopyMeshToSkeletalMesh` at `MeshAssetFunctions.h:153,166` on 5.3 (under
  `Plugins/Experimental/`) and `:195,208` on 5.4 (under `Plugins/Runtime/`);
  `CreateNewSkeletalMeshAssetFromMesh` at `CreateNewAssetUtilityFunctions.h:166` on 5.3. No
  `UE_VERSION_*` branching needed — only the existing plugin-directory probe at
  `PinWrightGeometry.Build.cs:28-32`, which already handles the 5.3 `Experimental` → 5.4+ `Runtime`
  move.
- **Size: ~350-500 lines, 1-2 new handler `.cpp` under
  `Source/PinWrightGeometry/Private/Handlers/Geometry/`**, plus the `docs/wiki-src/geometry.md`
  overlay and one test file. Each verb is param plumbing + one engine call + result JSON;
  `MeshAssetIOHandler.cpp` is the template and runs ~420 lines for one verb with LOD-type parsing,
  so budget generously.

**The pipeline is already proven on this host project's own content.** The raw-Python equivalents
produced all four shipped `SKM_Creep_*` assets: `copy_mesh_from_skeletal_mesh` →
`compute_smooth_bone_weights` → `copy_mesh_to_skeletal_mesh`, every call recording `SUCCESS` /
`saved: true`. Every skeletal read used `GeometryScriptLODType.SOURCE_MODEL`; every write-back used
`GeometryScriptBoneHierarchyMismatchHandling.DO_NOTHING`. These are the settings the verbs should
default to.

## Hazards to check during implementation

- **Modal UI on the game thread.** Board precedent `B-physics-asset-factory-modal-hang` (IN-REVIEW):
  `UPhysicsAssetFactory` hangs the editor with a modal dialog. Verify
  `CreateNewSkeletalMeshAssetFromMesh` opens no UI under `-unattended`.
- **Bone-weight ordering.** `mesh_create_bone_weights` must run *after* all geometry is appended or
  the asset fails to create — see `E-geometry-script-skeletal-authoring-traps`. A composite
  bind-weights verb should enforce the ordering rather than document it.
- `GeometryScriptCreateNewSkeletalMeshAssetOptions.materials` is a `TMap<FName, UMaterialInterface*>`,
  not a list (the StaticMesh options struct has no `materials` field at all — set
  `StaticMesh.static_materials` afterwards there).

## Follow-on group

A second group of ~5-6 bone-weight verbs (`copy_bones_from_skeleton`, `mesh_create_bone_weights`,
`set_vertex_bone_weights`, `compute_smooth_bone_weights`, `get_vertex_bone_weights`,
`mesh_has_bone_weights`) is the natural next step, ideally folded into one composite
`bind_skin_weights` verb so the ordering trap is enforced in code. Filed here as context, not as
part of this ticket's scope.

severity rationale: impact=hard blocker with no RPC workaround — a whole authoring domain is
unreachable and the gap is invisible until after commitment (`create_skeleton` exists, so the path
looks open) × reach=rare-to-moderate (skeletal authoring), offset upward because this is the single
highest-value gap found in the triage and it forced an entire project's work out of the typed API
-> High

## Relationship to other tickets

- `F-skeleton-no-mesh-for-physics-asset` (IN-REVIEW, Medium) — the *same* dead end, scoped narrowly
  to physics-asset reachability. Its accepted fix routed **around** the missing mesh (it taught
  `skeleton.create_physics_asset` to build bodies from a bare skeleton's reference pose) rather than
  providing one, so the underlying "no verb makes a `USkeletalMesh`" hole is untouched and this
  ticket is not closed by it.
- `E-geometry-namespace-skeletal-scope-undocumented` — the discoverability half: the ~90 `geometry.*`
  verbs advertise no scope boundary, so callers do not learn about this gap until they hit it. That
  ticket is the doc fix; this one is the capability.
- `B-weight-mutators-write-profile-not-base-skinning` (OPEN) — this round-trip is the viable path to
  writing **base** skin weights, which no `skeleton.*` verb can do.
- `B-geometry-convert-static-mesh-no-disk-write` (IN-REVIEW, Critical) and
  `B-convert-static-mesh-uvless-crash` (IN-REVIEW, Critical) — read both before adding an
  asset-creation verb to `geometry.*`; they are the existing failure modes in the neighbouring
  static-mesh creation path.

## History
- `#1-triage-no-skeletal-mesh-creation` `OPEN` reporter — Found during a mesh/skeletal authoring triage at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`. Confirmed by grep that `NewObject<USkeletalMesh>` occurs only in the seven `Private/Tests/` locations listed in the body, and that `SkeletalMeshFactory` / `CreateNewSkeletalMeshAssetFromMesh` have zero hits anywhere in `Source/` — no verb in any namespace creates a `USkeletalMesh` or writes skeletal geometry. `skeleton.create_skeleton` (`SkeletonHandler.cpp:486`) does exist alongside 41 other `skeleton.*` verbs that edit existing assets, so the namespace reads as complete and the dead end is only discovered after an agent has authored a bare `USkeleton` it can never give a mesh to. Feasibility checked, not assumed: `GeometryScriptingCore` and `GeometryScriptingEditor` are already linked at `PinWrightGeometry.Build.cs:54`, and all three needed entry points exist on UE 5.3-5.8 (`MeshAssetFunctions.h:153,166` on 5.3 / `:195,208` on 5.4; `CreateNewAssetUtilityFunctions.h:166` on 5.3), so this needs **no `Build.cs` change and no version gating** — roughly 3 verbs, ~350-500 lines, 1-2 new handler files. The pipeline is already proven on this host project's content: the raw-Python `copy_mesh_from_skeletal_mesh` → `compute_smooth_bone_weights` → `copy_mesh_to_skeletal_mesh` chain produced all four shipped `SKM_Creep_*` meshes with `SUCCESS`/`saved:true` on every call, which is exactly why all skeletal authoring here left the typed API for `python.execute`. Not closed by `F-skeleton-no-mesh-for-physics-asset` (IN-REVIEW), whose fix routed around the missing mesh instead of providing one.
- `#2-landed-in-4ba04d5b` `IN-REVIEW` developer — Implemented, by an earlier pass, not by this one: plugin commit `4ba04d5b` (Add a skeletal-mesh round trip to the typed geometry.* surface) added `geometry.create_from_skeletal_mesh`, `geometry.bind_skin_weights` and `geometry.convert_to_skeletal_mesh` in `Source/PinWrightGeometry/Private/Handlers/Geometry/SkeletalMeshAssetIOHandler.cpp`. The ticket's follow-on suggestion to fold the ~6 bone-weight verbs into one composite `bind_skin_weights` that ENFORCES the geometry-before-weights ordering was taken: the composite does copy-bones -> create-weights -> smooth-bind in one call, and `convert_to_skeletal_mesh` runs an exhaustive per-vertex coverage scan that rejects with `NO_SKIN_WEIGHTS` / `SKIN_WEIGHTS_INCOMPLETE` (see `E-geometry-script-skeletal-authoring-traps`). Status moved OPEN -> IN-REVIEW only to reflect that the code exists; this pass did not touch `Source/PinWrightGeometry/`. Recorded here because a separate triage was still reading this as OPEN and concluding that nothing can write base skin weights: `convert_to_skeletal_mesh` with `overwrite:true` calls `CopyMeshToSkeletalMesh` (`SkeletalMeshAssetIOHandler.cpp:1102-1103`), which rewrites the LOD's MeshDescription in place — and MeshDescription skin weights are BASE skinning. So this round trip IS the base-weight write path, and `skeleton.describe_skin_weights` now reports `baseSkinning` so its result is verifiable (`B-weight-mutators-write-profile-not-base-skinning`). Tester should verify the three verbs against a real asset before DONE.
