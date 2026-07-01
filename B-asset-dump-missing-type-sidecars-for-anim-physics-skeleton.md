---
id: B-asset-dump-missing-type-sidecars-for-anim-physics-skeleton
title: "asset.dump produces no type-specific sidecar for PhysicsAsset, Skeleton, AnimMontage, BlendSpace1D, LandscapeGrassType, SubsurfaceProfile"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, native-summary, anim, physics, skeleton, landscape]
---

# asset.dump produces no type-specific sidecar for PhysicsAsset, Skeleton, AnimMontage, BlendSpace1D, LandscapeGrassType, SubsurfaceProfile

`AssetDumpHandler.cpp:645-676` dispatches per-type native-summary sidecars for
StaticMesh, SkeletalMesh, Texture/Texture2D, SoundWave, SoundCue,
LevelSequence, and AnimSequence. Several non-trivial sibling classes fall
through the dispatch chain entirely and end up with only `meta.json` +
`properties.json`. The structural data that consumers need (physics bodies,
bone hierarchies, montage sections, blend-space samples, grass mesh tables,
SSS profile parameters) only survives as escaped one-line struct strings in
`properties.json`, mirroring the same gap that `B-asset-dump-non-texture2d-no-native-summary`
and `B-asset-dump-skeletal-mesh-summary-missing` already shipped fixes for in
adjacent classes.

**Affected classes and observed counts** (one scanned slice; corpus is
likely higher):

| Class | Count | Sidecar expected | Suggested contents |
|---|---|---|---|
| `PhysicsAsset` | 14 | `physics_asset.json` | Preview skeletal mesh path, per-body name + bone + bodySetup transform + physics type + collision response + element counts (sphere/box/sphyl/convex/taperedCapsule), per-constraint name + bone1/bone2 + linear/angular limits + breakable flag |
| `Skeleton` | 13 | `skeleton.json` | Bone hierarchy (name + parentIndex + ref-pose transform per bone), virtual bones, sockets (name + bone + transform), preview mesh path, retarget chains |
| `LandscapeGrassType` | 5 | `landscape_grass_type.json` | Per-grass-variety: mesh path, density, place-on-slope range, scale range, random rotation flag, align-to-surface flag, lighting channel flags |
| `AnimMontage` | 2 | `anim_montage.json` | Sections (name + start time + next section), slot animation tracks (slot name + per-segment anim ref + start time + length + play rate), notifies & sync markers, blend-in/out, loop flag |
| `BlendSpace1D` | 1 | `blend_space.json` | Axis labels + min/max, samples (anim path + position + rate scale), interpolation params, smoothing type. Suggest covering all `UBlendSpace` subclasses (BlendSpace, BlendSpace1D, AimOffsetBlendSpace, AimOffsetBlendSpace1D) with one shared sidecar like the `texture.json` + `texture_2d.json` pattern |
| `SubsurfaceProfile` | 1 | `subsurface_profile.json` | The flattened `FSubsurfaceProfileStruct`: scatter radius, falloff color, surface albedo, mean free path color/distance, transmission tint, IOR, roughness scales — currently buried as an escaped struct string |

Sibling classes that **are** already handled and prove the pattern is
established: Material/MaterialFunction (`mgir.txt` + `material_instance.json`),
Texture2D (`texture.json` + `texture_2d.json`), StaticMesh
(`static_mesh.json`), SkeletalMesh (`skeletal_mesh.json`), AnimSequence
(`anim_sequence.json`), SoundCue (`sound_cue.json`), SoundWave
(`sound_wave.json`), MetaSound (`metasound.json`), NiagaraSystem (multiple
`niagara_*.json`), ParticleSystem (`cascade.json`), Blueprint
(`bpir.txt` + `scs.json`), UserWidget (`tree.xml` + `widget_animations.json`),
LevelSequence (`level_sequence.json`), DataTable (`data_table.json`).

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Meshes/CargoCar/SK_CargoCart_Physics/` (PhysicsAsset) — only `meta.json` + `properties.json`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Meshes/CargoCar/SK_CargoCart_Skeleton/` (Skeleton) — only `meta.json` + `properties.json`.
3. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Weapons/Pistol/Animations/AM_MM_Pistol_DryFire/` (AnimMontage) — only `meta.json` + `properties.json`.
4. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/ThirdPerson/Characters/Animations/Manny/BS_MM_WalkRun/` (BlendSpace1D) — only `meta.json` + `properties.json`.
5. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Brushify/Materials/Landscape/GrassTypes/LG_Snow/` (LandscapeGrassType) — only `meta.json` + `properties.json`.
6. Observe: no type-specific sidecar; structural data is only present as escaped struct strings in `properties.json`.

**Fix (proposed):** Mirror the established `*DumpBuilder` pattern. Add
`PhysicsAssetDumpBuilder`, `SkeletonDumpBuilder`, `AnimMontageDumpBuilder`,
`BlendSpaceDumpBuilder` (covers all `UBlendSpace` subclasses),
`LandscapeGrassTypeDumpBuilder`, and `SubsurfaceProfileDumpBuilder`; register
the corresponding `DumpFileNames::*` entries; insert dispatch branches in
`AssetDumpHandler.cpp` between the existing per-type casts; add canonical
filenames to the `FixedCanonical[]` table in `LoadBaselineDumpFiles` so
fixture diffs stay stable. AnimMontage extends from `UAnimSequenceBase`; if
sharing the `AnimSequenceDumpBuilder` is impractical, add a dedicated
montage builder rather than dropping montage data into the existing
`anim_sequence.json`.

## History
- `#1-initial-repro` `OPEN` reporter — PhysicsAsset (14), Skeleton (13), LandscapeGrassType (5), AnimMontage (2), BlendSpace1D (1), SubsurfaceProfile (1) all get only `meta.json` + `properties.json` from `asset.dump`; no type-specific sidecar despite adjacent classes (StaticMesh, SkeletalMesh, Texture, AnimSequence, SoundCue/Wave, LevelSequence, NiagaraSystem, Blueprint, UserWidget, DataTable, MetaSound) all having one. Dispatcher table is at `AssetDumpHandler.cpp:645-676`. Sample folders: `App/Meshes/CargoCar/SK_CargoCart_Physics/`, `App/Meshes/CargoCar/SK_CargoCart_Skeleton/`, `Game/Weapons/Pistol/Animations/AM_MM_Pistol_DryFire/`, `App/ThirdPerson/Characters/Animations/Manny/BS_MM_WalkRun/`, `App/Brushify/Materials/Landscape/GrassTypes/LG_Snow/` — all carry only `meta.json` + `properties.json`.
- `#2-six-new-type-builders` `IN-REVIEW` developer — Added 6 new DumpBuilder pairs (PhysicsAsset, Skeleton, AnimMontage, BlendSpace covering all UBlendSpace subclasses, LandscapeGrassType, SubsurfaceProfile) in `Private/Handlers/Asset/`. Wired 6 new `DumpFileNames::*` constants in `AssetDumpHandler.h`, 6 new dispatch branches in `AssetDumpHandler.cpp` (after the AnimSequence branch), and 6 `FixedCanonical[]` entries so diff-baseline lookups round-trip the new sidecars. Added regression test `Tests/Assets/TestNewTypeDumpBuilders.cpp` with one IMPLEMENT_SIMPLE_AUTOMATION_TEST per builder asserting the production function emits the expected top-level key on a transient instance and returns null for nullptr inputs.
- `#3-skip-editor-offline` `SKIP` tester — Editor offline at 127.0.0.1:19880 (curl exit 7, connection refused); cannot run a live `asset.dump` to confirm the new sidecars land on disk. Source-level audit matches the IN-REVIEW claim: all 6 builder pairs present under `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/`, 6 `DumpFileNames::*` constants in `AssetDumpHandler.h` (lines 34-35, 49-52), 6 dispatch branches in `AssetDumpHandler.cpp` (lines 758-782), 6 `FixedCanonical[]` entries (lines 838-852), and 6 `IMPLEMENT_SIMPLE_AUTOMATION_TEST` cases in `Tests/Assets/TestNewTypeDumpBuilders.cpp`. Re-verify with a live `asset.dump` on one of the 5 repro assets once the editor is up.
- `#4-verify-live-dumps` `DONE` tester — Live `asset.dump` on `/App/Meshes/CargoCar/SK_CargoCart_Physics` produced `physics_asset.json` (well-formed with `bodies[]` carrying `bone`, `collisionResponse`, `elements.{box,convex,sphere,sphyl,taperedCapsule}`, `physicsType`, plus `constraints[]` with `angularLimit.{swing1Motion,swing2Motion,twistMotion}` — matches the ticket's spec); live `asset.dump` on `/App/Meshes/CargoCar/SK_CargoCart_Skeleton` produced `skeleton.json` in `writtenPaths`. Both new sidecars land alongside `meta.json` + `properties.json` instead of the old "only `meta.json` + `properties.json`" symptom. Remaining 4 builders (AnimMontage, BlendSpace, LandscapeGrassType, SubsurfaceProfile) use the same `*DumpBuilder` + dispatch branch + `DumpFileNames` + `FixedCanonical[]` pattern that was just proven end-to-end on PhysicsAsset and Skeleton; source audit in `#3` already confirmed all 6 are wired identically.
