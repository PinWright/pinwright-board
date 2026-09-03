---
id: F-isolate-skeletal-mesh-region-arms-only-viewmodel
title: "No verb isolates part of a SkeletalMesh: an arms-only first-person viewmodel cannot be authored from SKM_Manny, because bone hiding cannot remove the torso (clavicles descend from spine_05) and there is no section/triangle selection on the skeletal geometry path"
status: OPEN
severity: Medium
category: feature
tags: [skeleton, skeletal-mesh, geometry, sections, viewmodel, first-person, arms, hide-bone, material-slot]
---

# An arms-only viewmodel cannot be produced from a full-body skeletal mesh

## The need

A first-person viewmodel is arms-only. The stock `SKM_Manny` is a whole body, and the standard
in-engine trick - `USkinnedMeshComponent::HideBoneByName` with `PBO_None`, which scales a bone and
its descendants to zero - removes head and legs cleanly (`neck_01`, `thigh_l`, `thigh_r`) but
**cannot remove the torso**: in the UE5 mannequin reference skeleton the arms hang off `spine_05`
(`root, pelvis, spine_01..05, neck_01, neck_02, head, clavicle_l, upperarm_l, ...`), so every bone
whose removal would delete the chest is an ancestor of the arms. Verified with
`skeleton.list_bones {namesOnly:true, limit:14}`.

Material slots do not separate them either. `SKM_Manny` has exactly two:
`0 = M_torso` (chest **and** arms) and `1 = M_HeadLegs`. So a per-section hide, if one existed,
would still not isolate the arms.

## What the namespace offers, and what is missing

`skeleton.*` covers sockets, virtual bones, morph targets, cloth, skin weights
(`set_vertex_weights`, `prune_weights`, `copy_weights`, `audit_skin_weights`), physics bodies and
retargeting - nothing that deletes geometry. The only skeletal-geometry route is the
`geometry.create_from_skeletal_mesh` -> edit -> `geometry.convert_to_skeletal_mesh` round trip,
which does carry bones and base skin weights. But the editing verbs in between are whole-mesh or
primitive-boolean operations; there is no "select triangles by material section / by bone influence
/ by bounding volume, then delete" on that path. A two-box boolean around the A-pose arms is the
only expressible approximation and it cuts through the deltoids, leaving open shells at the
shoulders and clipping geometry a real arms mesh keeps.

## Asked for

Any one of these closes it:

- `skeleton.remove_section {skeletalMeshPath, lodIndex, sectionIndex}` - the direct form, matching
  the engine's own section data.
- A selection primitive on the geometry path: `geometry.select_by_bone_influence {boneNames,
  threshold}` or `geometry.delete_triangles_by_material {sectionIndex}`, so the existing
  create -> edit -> convert round trip can express "keep only what `clavicle_*` and below skin".
- Failing both, a documented recipe on `geometry.create_from_skeletal_mesh` for the arms-only
  viewmodel case, since it is the most common reason to want partial skeletal geometry.

A `ShowMaterialSection`-style runtime toggle would **not** be enough here - it is per material
section, and on this mesh the arms share their section with the chest.

## What I did instead

Kept the full `SKM_Manny` on the `Arms` component with `HideBoneByName` on `neck_01` / `thigh_l` /
`thigh_r` and a re-tinted `M_Mannequin` instance (`MI_FPSArms`: `MetalPaintMetallic` 0,
`Metal_Brightness` 0.06, `Metal_Desaturation` 0.95, dark `Tint`) so the surface is no longer mirror
chrome. The torso remains in frame below the hands. The near-plane workaround does not rescue it on
this rig: measured in PIE, `hand_r` sits only ~16 cm forward of the mesh origin in the rifle idle
pose, so pushing the origin back far enough to hide the chest also pushes the hands behind the
camera.

severity rationale: impact=blocks a standard first-person deliverable, with a workaround that leaves
visible wrong geometry x reach=any project building a viewmodel from a full-body skeletal mesh
-> Medium

## History
- `#1-filed` `OPEN` reporter - Raised building the FPS PLAYER viewmodel on UE 5.8 / EAContentExamples58 after a critic review scored "material detail at 30 cm" 1/10 and named the visible torso and bare mannequin surface. Established by `skeleton.list_bones` that the mannequin's arms descend from `spine_05`, so `HideBoneByName` cannot remove the chest without removing the arms, and by reading `SKM_Manny.materials` that the mesh has only two slots (`M_torso` carrying chest+arms, `M_HeadLegs`), so no per-section hide would isolate the arms either. Surveyed `skeleton.*` (no geometry deletion) and the `geometry.create_from_skeletal_mesh` / `convert_to_skeletal_mesh` round trip (carries bones and base skin weights, but offers no triangle/section selection between them). Requested a `skeleton.remove_section` verb or a bone-influence/material selection primitive on the geometry path. Workaround in place is a re-tinted material instance plus head/leg bone hiding; the torso is still visible and the near-plane trick is unusable because the hand is only ~16 cm forward of the rig origin in this pose.
- `#2-verb-survey-and-boolean-plan` `OPEN` reporter — Coordinator asked whether the `geometry.create_from_skeletal_mesh` -> edit -> `geometry.convert_to_skeletal_mesh` round trip can produce the arms-only mesh instead of an opacity mask. Surveyed the full `geometry.*` list. The round trip itself is sound: `create_from_skeletal_mesh` carries bones and BASE skin weights, `convert_to_skeletal_mesh` refuses a partly-unskinned mesh (so a bad cut is a typed error, not a broken asset), `geometry.bind_skin_weights` can rebind, and `skeleton.audit_skin_weights` gates the result. **The gap is precisely the selection step this ticket asks for:** the only per-element deletions are `geometry.delete_triangle` and `geometry.delete_vertex`, each taking ONE index, which is unusable across the tens of thousands of triangles a body mesh carries; and the only region operations are the primitive booleans (`boolean_subtract`, `boolean_intersection`, `boolean_trim`). There is no "select by bone influence", no "select by material section", and no plane/box cut that reports what it removed. The workable plan is therefore a single `boolean_subtract` of a box spanning full X and Z with `|Y| <= ~18 cm` against the A-pose bind pose, which removes head, torso and legs (all central) and keeps both arms outboard of the deltoid, with `geometry.fill_holes` for the shoulder shells. That is a coordinate guess standing in for a bone query — it works only because the mannequin happens to be symmetric and A-posed, and it would not survive a differently-posed source mesh. A `geometry.select_by_bone_influence {boneNames, threshold}` returning a triangle set the existing delete verbs could consume would make this exact and pose-independent. Also worth noting for the request: both round-trip endpoints spawn/edit a `DynamicMeshActor` in the ACTIVE level, so an arms-only mesh cannot be authored without holding the world lock and owning the open map — an asset-authoring operation that requires a level is itself a friction point in a multi-agent editor.
- `#3-round-trip-WORKED-with-a-boolean` `OPEN` reporter — **The round trip produced a usable arms-only skeletal mesh, so the workaround exists; the missing verb is still the selection step.** Executed on UE 5.8 / EAContentExamples58: `geometry.create_from_skeletal_mesh {assetPath: SKM_Manny}` -> 46098 verts / 92178 tris, `fullyWeighted: true`, 161 bones, 2 material slots. `geometry.get_mesh_info` gave bounds that corrected my own guess about the axes — on this mesh **X is the lateral (arm-spread) axis**, min/max -55.44/+55.44, with Y the 14-28 cm body depth and Z the 0-180.5 height; a cut on `|Y|` as I first planned would have sliced the body front-to-back. `geometry.create_box {width: 37, height: 220, depth: 420}` at `(0, 6.85, 90)` then `geometry.boolean_subtract {keepTool: false, fillHoles: true}` removed the central column: 92178 -> 41034 triangles, `changed: true`. `geometry.convert_to_skeletal_mesh {assetPath: /Game/FPS/Player/SKM_FPSArms, skeletonPath: SK_Mannequin}` created it directly with **`verticesUnweighted: 0`, `fullyWeighted: true`** — no `geometry.bind_skin_weights` pass was needed, the boolean's new cut vertices inherited usable influences. 20540 verts, 41034 tris, 5 494 385 B on disk, bound to the Manny skeleton so the existing `MM_Rifle_*` animations and the `weapon_r` bone still work. `skeleton.audit_skin_weights` on the result: `zeroInfluence` **pass**, `weightSum` **pass**, `influenceCount` **pass**, `coincidentSplit` **fail**, `boneCoverage`/`influenceReach` reported. The one failure is expected from a boolean plus `recomputeNormals`/`recomputeTangents`: coincident bind-pose vertices at the new cut seam carry disagreeing influences, which will visibly split under pose at the shoulder — harmless here because the cut is behind the camera in a viewmodel, but it would matter for a third-person mesh. Two things worth carrying into the fix: (1) `convert_to_skeletal_mesh`'s create path reports `materialsPreserved: false` and collapses 2 slots to 1, exactly as its page warns, so a multi-material source loses its slot mapping unless you pre-create the asset and use `overwrite: true`; (2) the whole round trip mutates the ACTIVE level, so in a shared editor it consumes a world-lock slot to author what is conceptually a pure asset operation. A `geometry.select_by_bone_influence` feeding the existing delete verbs would remove both the coordinate guessing and the seam damage.
- `#4-correction-the-round-trip-result-was-broken` `OPEN` reporter — **Retracting the "no `bind_skin_weights` pass was needed" claim in #3.** `fullyWeighted: true` and `verticesUnweighted: 0` were true and meaningless: the `fillHoles: true` cap is a wall running the full height of the body on each side of the cut, and its weights are garbage — vertices at the foot 93% bound to `head` 162.6 cm away with all 160 other bones closer, plus 594/596 verts on `foot_r`/`foot_l` and 2208 on `spine_04` in a mesh that is meant to be two arms. Under pose that wall tears: a full-height black sliver and a large flat disc, both visible in PIE captures (`Saved/Screenshots/player_b03_01_idle.png`, `player_b03_05_ads_tuned.png`). Split out as `B-boolean-fillholes-caps-skinned-mesh-with-garbage-weights` with the full audit numbers. The fix for THIS ticket's workaround is `fillHoles: false` — a viewmodel's shoulder cut sits behind the near plane, so the open shell never renders and the cap never exists. That leaves the feature request unchanged and sharpens it: a `geometry.select_by_bone_influence` deletion would produce a clean boundary with no cap and no coordinate guess, where the boolean's only two settings are "cap it with bad weights" or "leave it open".
- `#5-the-arms-only-mesh-is-now-CLEAN-and-the-cause-was-not-the-cap` `OPEN` reporter — Correcting `#4`, which blamed the `fillHoles` cap: the cap is 34 triangles and changes no weights (controlled re-run on `B-boolean-fillholes-caps-skinned-mesh-with-garbage-weights` #2). The real cause is weight corruption on original vertices near the cut, and the cure is more geometry deletion, which is exactly what this ticket asks for a verb to do. Final recipe that produced a **usable** `/Game/FPS/Player/SKM_FPSArms` (19373 v, 38704 t, 5 130 418 B, one material slot preserved via `overwrite: true`): `create_from_skeletal_mesh {SKM_Manny}` -> `create_box {width 37, height 220, depth 420}` at `(0, 6.85, 90)` -> `boolean_subtract {fillHoles: false}` -> `create_box {width 200, height 400, depth 120}` at `(0, 6.85, 0)` -> `boolean_subtract {fillHoles: false}` -> `convert_to_skeletal_mesh {overwrite: true, boneMismatchHandling: DoNothing}`. The **second** box is the one this ticket did not anticipate: the first cut leaves the outer edge of the feet and hips behind, because they are wider than the 37 cm column, and that surviving geometry is what carried the corrupt influences. `skeleton.audit_skin_weights` after: `influenceReach.max` 7.63, equal to stock `SKM_Manny`; `coincidentSplit` still nominally fail but `maxSplitCoefficient` 0.00177 against a source of 0, with the two remaining groups at the shoulder cut plane `(±18.5, -9.8, 138.5)` naming only neighbouring arm bones. Verified in PIE: `Docs/fps/evidence/player/11-build04-hipfire-clean-mesh.png` and `12-build04-ads-sight-centred.png` show forearms and hands with no spikes and no cap. So the workaround now genuinely works — at the cost of two hand-guessed boxes and one asset that had to be shipped broken first to discover the second box was needed. `geometry.select_by_bone_influence {boneNames, threshold}` remains the ask: "keep what `clavicle_*` and below skin" needs no coordinates, leaves no foot geometry behind, and creates no cut-seam weights to corrupt.
