---
id: B-weight-mutators-write-profile-not-base-skinning
title: "All four skeleton weight mutators write a named skin-weight PROFILE, never base section skinning — while documenting themselves as writing 'skin weights'"
status: IN-REVIEW
severity: High
category: bug
tags: [skeleton, skin-weights, skeletal-mesh, alternate-profile, mislabelled-verb, capability-gap]
---

# skeleton weight mutators write an alternate profile, not base skin weights

Every skin-weight mutator in the `skeleton.*` namespace writes a **named skin-weight profile** —
what the engine calls an *alternate influence* set — and never touches the LOD's base section
skinning that the renderer uses by default. The verbs are documented as writing "skin weights",
which callers reasonably read as base skinning. They are not the same thing, and nothing reads the
alternate set unless a component explicitly activates that profile.

This is architectural, not an index-math bug. It is **not** fixed by
`B-skin-weight-transfer-writes-section-local-bone-indices`; that ticket corrects *which bone* an
influence points at inside the profile, this one is about the profile being the wrong destination
in the first place.

Line numbers as read at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`
(`Plugins/PinWright/`).

## Evidence

The single write path, `Source/PinWright/Private/Handlers/Animation/SkinWeightTransferUtils.h:473-491`:

```cpp
    inline void WriteSkinWeightProfile(
        USkeletalMesh& Mesh,
        FSkeletalMeshLODModel& LODModel,
        FName ProfileName,
        const TArray<FRawSkinWeight>& SkinWeights)
    {
        ...
            FSkinWeightProfileInfo NewProfile;
            NewProfile.Name = ProfileName;
            Mesh.AddSkinWeightProfile(NewProfile);
        ...
        FImportedSkinWeightProfileData& ProfileData = LODModel.SkinWeightProfiles.FindOrAdd(ProfileName);
        ProfileData.SkinWeights = SkinWeights;
        RebuildSourceModelInfluences(ProfileData.SkinWeights, ProfileData.SourceModelInfluences);
    }
```

`Mesh.AddSkinWeightProfile` + `LODModel.SkinWeightProfiles.FindOrAdd` is the alternate-profile
channel by definition. **`LODModel.Sections[*].SoftVertices` — the base skinning — is never
assigned by any handler path.**

The engine's own vocabulary confirms what this channel is:
`C:\UE_5.8\Engine\Source\Developer\SkeletalMeshUtilitiesCommon\Private\LODUtilities.cpp:2301` is
`FLODUtilities::UpdateAlternateSkinWeights`, and
`C:\UE_5.8\Engine\Source\Developer\MeshUtilities\Private\MeshUtilities.cpp:4206` comments
`//Get alternate skinning weights map to retrieve easily the data`.

All four verbs land here, each into its **own** default profile — so a caller running two of them
gets two unrelated alternate sets and still an untouched base mesh:

- `skeleton.normalize_weights` — registered `SkeletalMeshHandler.cpp:390`, default profile
  `NormalizedWeights` (`:409`)
- `skeleton.prune_weights` — registered `:449`, default `PrunedWeights` (`:479`)
- `skeleton.set_vertex_weights` — registered `:524`, default `CustomWeights` (`:538`)
- `skeleton.copy_weights` — registered `:686`, default `CopiedWeights` (`:701`)

The first three go through the shared scaffold `ApplyWeightEditToProfile`
(`SkeletalMeshHandler.cpp:159-199`); `copy_weights` duplicates the persist logic inline
(`:761-767`).

## Impact

A caller asks the tool to fix a mesh's skinning, gets `ok:true` plus a plausible
`verticesNormalized` / `verticesCopied` count, saves the asset — and the mesh renders exactly as
before, because nothing reads the profile that was written. There is no error, no warning, and no
field in the result that says "this went to an alternate profile, base skinning is untouched".

Compounding it: the caller's natural verification path, `skeleton.describe_skin_weights`
(`SkeletalMeshHandler.cpp:249`), reads **only named profiles** — its summarizer
`SkinWeightTransferUtils::SummarizeSkinWeights` (`SkinWeightTransferUtils.h:384`) is fed from
`LODModel.SkinWeightProfiles`. So the readback confirms the profile the mutator just wrote and
reports nothing about the base skinning the caller believed it was editing. The write and the
verification agree with each other and both disagree with the rendered mesh.

## Writing base weights is not reachable through this API at all

There is no partial fix inside `SkinWeightTransferUtils`. `SourceModelInfluences` +
`FSkinWeightProfileInfo` *is* the alternate channel. Changing base skinning means either:

- the import-data pipeline — `FSkeletalMeshImportData::Influences` + `SaveLODImportedData` + a
  rebuild — a different pipeline entirely; or
- **Geometry Script**: `CopyMeshFromSkeletalMesh` → edit weights on the DynamicMesh →
  `CopyMeshToSkeletalMesh`. This is the viable path, and it is the one this host project actually
  used successfully on four shipped `SKM_Creep_*` meshes
  (`copy_mesh_from_skeletal_mesh` / `compute_smooth_bone_weights` / `copy_mesh_to_skeletal_mesh`,
  all recorded `SUCCESS`). No typed verb exposes it — tracked as
  `F-geometry-skeletal-mesh-roundtrip-verbs`.

**The approach is not wrong for what it does; it is mislabelled.**

## Fix

Two parts, and the cheap one should not wait for the expensive one:

1. **Stop mislabelling (cheap, ~15 lines).** The four verbs' summaries must say *skin-weight
   profile*, not *skin weights*, and state explicitly that base section skinning is untouched and
   that the profile must be activated on a component to have any visible effect. Touch points:
   the four `REGISTER_RPC_HANDLER` summaries in
   `Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp` (`:390`, `:449`, `:524`,
   `:686`) and the `docs/wiki-src/skeleton.md` overlay. Echoing `profileName` back in each result
   (several already do) plus a `writesBaseSkinning:false`-style note would make it observable, not
   just documented.
2. **Provide a real base-weight path** — the Geometry Script round-trip, via
   `F-geometry-skeletal-mesh-roundtrip-verbs`.

severity rationale: impact=silent false-success on a normal path (the caller is told skin weights
were written, believes the mesh was fixed, and builds on it; the only readback agrees with the lie)
+ hard blocker with no RPC workaround for base skinning at all × reach=rare (skin-weight authoring),
but it is the *entire* weight-mutator family — 4 of the 5 weight verbs in the namespace -> High

## Relationship to other tickets

- `B-skin-weight-transfer-writes-section-local-bone-indices` (OPEN) — the index-space bug *inside*
  the profile these verbs write. Independent and narrower: fixing it yields a correct alternate
  profile, still no base skinning. Fix them separately; do not conflate.
- `B-skeleton-normalize-recaptures-base-clobbers-profile` (IN-REVIEW) — that ticket made the
  mutators *seed* from the authored profile rather than from base skinning. It is about the read
  side of the same scaffold; the write side has always been profile-only and remains so.
- `F-skeleton-skin-weight-profile-readback` (IN-REVIEW) — added `describe_skin_weights`, which is
  also profile-only, so it cannot surface this discrepancy.
- `F-geometry-skeletal-mesh-roundtrip-verbs` — the proposed real base-weight path.

## History
- `#1-triage-write-target-is-alternate-profile` `OPEN` reporter — Found during a mesh/skeletal authoring triage at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`. Source-confirmed that the sole persist path `SkinWeightTransferUtils::WriteSkinWeightProfile` (`SkinWeightTransferUtils.h:473-491`) writes only `USkeletalMesh::AddSkinWeightProfile` + `FSkeletalMeshLODModel::SkinWeightProfiles.FindOrAdd`, and that `LODModel.Sections[*].SoftVertices` is never assigned on any handler path — so all four weight mutators (`normalize_weights` `:390`/`NormalizedWeights` `:409`, `prune_weights` `:449`/`PrunedWeights` `:479`, `set_vertex_weights` `:524`/`CustomWeights` `:538`, `copy_weights` `:686`/`CopiedWeights` `:701`) write an engine *alternate influence* set that nothing reads by default, while their docs say "skin weights". Engine vocabulary cross-checked: `FLODUtilities::UpdateAlternateSkinWeights` (`UE_5.8 .../LODUtilities.cpp:2301`) and the "alternate skinning weights map" comment at `MeshUtilities.cpp:4206`. The masking is closed-loop: the only readback, `skeleton.describe_skin_weights` (`SkeletalMeshHandler.cpp:249`, summarizer `SkinWeightTransferUtils.h:384`), reads profiles only, so it confirms the write and says nothing about base skinning. Writing base weights is not reachable through this API — it needs the import-data pipeline or Geometry Script `CopyMeshFromSkeletalMesh` → weights → `CopyMeshToSkeletalMesh`, the latter being the path this host project used successfully on four shipped `SKM_Creep_*` meshes. Architectural: explicitly NOT fixed by `B-skin-weight-transfer-writes-section-local-bone-indices`.
- `#2-honest-verbs-and-base-readback` `IN-REVIEW` developer — Resolved as a LABELLING + READBACK fix, not by repointing the mutators at base skinning. Finding that decided the shape: base skin weights ARE already writable as of plugin commit `4ba04d5b` — `geometry.convert_to_skeletal_mesh` with `overwrite:true` calls `UGeometryScriptLibrary_StaticMeshFunctions::CopyMeshToSkeletalMesh` (`Source/PinWrightGeometry/Private/Handlers/Geometry/SkeletalMeshAssetIOHandler.cpp:1102-1103`), which rewrites the LOD's MeshDescription in place while preserving material slots, and MeshDescription skin weights ARE base skinning (`FSkeletalMeshAttributes::GetVertexSkinWeights(NAME_None)`, UE 5.8 `SkeletalMeshAttributes.h:294`, default profile name at `:95`). What did not exist was any way to SEE that it landed: `skeleton.describe_skin_weights` was documented as the mutators' verification counterpart (`Docs/wiki-src/skeleton.md:125-127`) and read only `LODModel.SkinWeightProfiles`, so the write path and the check path agreed with each other while both disagreed with the mesh — and on a mesh with no authored profile (every FBX import, every `geometry.convert_to_skeletal_mesh` output) it answered `profileCount:0` with a success status, having measured nothing. Changes: (a) `describe_skin_weights` now returns a `baseSkinning` block read from `FSkelMeshSection::SoftVertices` with every influence resolved through the section `BoneMap` to a reference-skeleton index AND a bone name, alongside the profiles block now labelled `profileKind: alternateSkinWeightProfile`; every per-LOD block states its `boneIndexSpace`; unmappable slots are dropped, counted as `unmappedInfluences`, and reported as `boneName: <unmapped>` rather than resolved to bone 0. (b) A mesh with no imported LOD models now returns `NO_LOD_MODELS` instead of an empty success — the same rule `audit_skin_weights` follows; `includeBaseSkinning:false` restores the old profile-names-only behaviour. This is the one backward-incompatible change and belongs in the changelog. (c) All four mutators now emit `wrote: alternateSkinWeightProfile`, `baseSkinningModified: false` and a `verifyWith` pointer, and their summaries say plainly that base skinning is untouched. (d) `auto_skin_weights`'s rejection message previously pointed callers at normalize/prune/set_vertex_weights to change per-vertex influences, which was the same lie one level up; it now names the geometry round trip. (e) `Docs/wiki-src/skeleton.md` gained a `## Skin weights: base skinning vs. alternate profiles` section above the first `###` (so it renders on the namespace page) covering both channels, the three-call base-weight recipe, and the two bone index spaces. Tests: `PinWright.skeleton.describe_skin_weights.ReportsBaseSkinningNotJustProfiles` builds a fixture where the profile reports 2 normalized / 0 degenerate while base skinning reports 1 degenerate IN THE SAME RESPONSE, and `...ProfilelessMeshStillReportsBaseSkinning` covers the profileCount:0 case — both in the new `Source/PinWright/Private/Tests/Gameplay/TestSkinWeightBaseReadback.cpp`. DELIBERATELY NOT DONE: no new base-weight write verb. An in-place per-vertex writer would have to go through `USkeletalMesh::GetMeshDescription`/`CommitMeshDescription` (UE 5.8 `SkeletalMesh.h:538`, `:622`), whose vertices are IMPORT (DCC) points, not render vertices — 8 for a cube, not 24 (`SkeletalMeshLODImporterData.h:51-52`) — so it would introduce a SECOND unspecified index space into a namespace whose every other verb indexes render vertices. The mapping exists (`FSkeletalMeshLODModel::MeshToImportVertexMap`, `SkeletalMeshLODModel.h:333-336`) and is many-to-one, so that remains a real follow-up but a designed one. **NOT COMPILED, NOT RUN.**
