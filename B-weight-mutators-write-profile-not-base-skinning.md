---
id: B-weight-mutators-write-profile-not-base-skinning
title: "All four skeleton weight mutators write a named skin-weight PROFILE, never base section skinning — while documenting themselves as writing 'skin weights'"
status: OPEN
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
