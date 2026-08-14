---
id: B-skin-weight-transfer-writes-section-local-bone-indices
title: "skeleton.normalize_weights / prune_weights / set_vertex_weights / copy_weights write section-local bone indices into a reference-skeleton index field"
status: IN-REVIEW
severity: High
category: bug
tags: [skeleton, skin-weights, multi-section, index-space, silent-corruption]
---

# skeleton weight mutators write section-local bone indices as reference-skeleton indices

`SkinWeightTransferUtils::RebuildSourceModelInfluences` (`SkinWeightTransferUtils.h:443-462`)
writes `FVertInfluence::BoneIndex` straight from an `FRawSkinWeight` seeded by
`CaptureBaseSkinWeights`, which is itself seeded from `FSkeletalMeshLODModel::GetVertices()`.

Those vertex influence slots are **section-local indices into `FSkelMeshSection::BoneMap`**, not
reference-skeleton bone indices. `FSkeletalMeshLODModel::GetVertices`
(`SkeletalMeshLODModel.cpp:1012-1053`) memcpys section vertices verbatim unless it is handed an
`FSkeletalMeshUnifiedBoneIndices`, and the engine's own CPU skin path does the `BoneMap`
indirection explicitly at `SkinnedMeshComponent.cpp:6250-6256`. But `Build()` re-chunks
`FVertInfluence::BoneIndex` as a **reference-skeleton** index.

Consequence: on a mesh with more than one section, `skeleton.normalize_weights`,
`skeleton.prune_weights`, `skeleton.set_vertex_weights` and `skeleton.copy_weights` write a
skin-weight profile whose influences point at the **wrong bones**. There is no error and no
warning — the profile is produced and looks valid.

## Correction: it is FOUR mutators over TWO call paths, not three

This ticket originally named three mutators. Re-triaged at plugin HEAD
`9da255f6d0ef5017cfddee03cd9458266c99d764`; all line numbers below are as read at that commit.

Three verbs reach `RebuildSourceModelInfluences` through the shared scaffold
`ApplyWeightEditToProfile` (`SkeletalMeshHandler.cpp:159-199`, which calls
`SkinWeightTransferUtils::WriteSkinWeightProfile` at `:30` of that block):

- `skeleton.normalize_weights` — registered `SkeletalMeshHandler.cpp:390`, scaffold call at `:426`
- `skeleton.prune_weights` — registered `:449`, scaffold call at `:499`
- `skeleton.set_vertex_weights` — registered `:524`, scaffold call at `:576` ← **missing from the
  original list**

The fourth, `skeleton.copy_weights` (registered `:686`), **bypasses the shared scaffold entirely**
and calls the rebuild inline, `SkeletalMeshHandler.cpp:761-767`:

```cpp
    const int32 VerticesCopied = SkinWeightTransferUtils::CopyClosestVertexWeights(
        SourceVertices, TargetVertices, ProfileData.SkinWeights);

    // SourceModelInfluences feeds the re-chunking step on Build(); rebuild it from
    // the copied weights so the saved profile survives a re-chunk instead of being empty.
    SkinWeightTransferUtils::RebuildSourceModelInfluences(
        ProfileData.SkinWeights, ProfileData.SourceModelInfluences);
```

**This is acceptance-criteria-level.** A fix applied only inside `WriteSkinWeightProfile` /
`ApplyWeightEditToProfile` repairs three verbs and **silently misses `copy_weights`**, which keeps
writing section-local indices with no compile error and no test failure. Both call sites must be
updated: `SkinWeightTransferUtils.h:490` and `SkeletalMeshHandler.cpp:766`. The existing test call
site `TestAnimationHandlers.cpp:3242` also needs its signature updated if
`RebuildSourceModelInfluences` gains an `FSkeletalMeshLODModel` parameter.

Preferred shape: have `copy_weights` call `WriteSkinWeightProfile` outright instead of keeping its
own copy of the persist logic — that removes the drift risk permanently (and also fixes
`B-copy-weights-duplicates-profile-info`, which exists only because of this duplication).

## This bug is stacked on a more fundamental one

Fixing the index math does **not** make these verbs write skin weights. Every one of the four
writes a *named skin-weight profile* (an engine "alternate influence" set) and never touches the
LOD's base section skinning — `LODModel.Sections[*].SoftVertices` is never assigned on any path
(`SkinWeightTransferUtils.h:473-491`). Tracked separately as
`B-weight-mutators-write-profile-not-base-skinning`. That ticket is architectural and is not
addressed by this one; conversely this one is real and worth fixing on its own terms, because the
profile that *is* written is currently corrupt on any multi-section mesh.

Do not let a fix here be read as "the mutators now write correct skin weights" — after this fix
they write a *correct alternate profile*, and base skinning is still unwritable through any RPC.

Invisible on any single-section mesh, where the BoneMap is effectively the identity, which is why
it has not been noticed.

**This is not hypothetical in this project.** Runtime measurement during integration pass 4:
`skeleton.audit_skin_weights` on `/Game/DotaBlockout/Creeps/Meshes/SKM_Creep_R_Melee` reports
`sectionCount: 6` over 3191 vertices and 161 bones. Every generated creep mesh is multi-section,
so any weight-repair call against one would corrupt it silently.

Found by inspection while building `skeleton.audit_skin_weights` (which resolves the indirection
correctly and is pinned by `PinWright.skeleton.audit_skin_weights.ResolvesSectionBoneMap`, a
fixture with two sections whose slot 0 maps to different real bones — an implementation that skips
the indirection reports both as bone 0 and passes).

Deliberately NOT fixed in that pass: it means changing shipping mutators, and the audit verb
makes the symptom observable first.

Acceptance: **all four** mutators (`normalize_weights`, `prune_weights`, `set_vertex_weights`,
`copy_weights`) map each influence through its section's `BoneMap` before writing
`FVertInfluence::BoneIndex`, with a regression test on a two-section fixture whose sections have
disjoint BoneMaps. The test must exercise the `copy_weights` path specifically — a fix landed only
in the shared scaffold leaves `copy_weights` broken and a scaffold-only test still green.
`Source/PinWright/Private/Tests/Gameplay/TestSkinWeightAudit.cpp:109` already builds multi-section
`USkeletalMesh` fixtures; reuse that builder.

Implementation note: the engine hands you the exact primitive it uses itself —
`FSkeletalMeshLODModel::GetSectionFromVertexIndex(VertexIndex, OutSectionIndex,
OutSectionVertexIndex)` then `Section.BoneMap[localIndex]`, the pattern at
`C:\UE_5.8\Engine\Source\Developer\MeshUtilities\Private\MeshUtilities.cpp:5761-5762`. No
`Build.cs` change: all of it is in the already-linked `Engine` module, already included at
`SkeletalMeshHandler.cpp:19`.

Engine caveat for anyone re-deriving the contract (do **not** read it as a counter-argument):
`MeshUtilities.cpp:5772-5779`, inside `FMeshUtilities::CreateImportDataFromLODModel`, resolves
through `BoneMap` for `AlternateInfluence.Influences` but *not* for `SourceModelInfluences`, on
adjacent lines of the same loop. That is an inconsistency in a legacy-conversion path; the
authoritative contract is set by `FLODUtilities::UpdateAlternateSkinWeights`
(`LODUtilities.cpp:2336-2347`) plus `SkeletalMeshTools::ChunkSkinnedVertices`
(`MeshUtilities.cpp:4214-4226` → `SkeletalMeshTools.cpp:579`), both of which require RefSkeleton
indices.

Migration risk: any profile authored by the *current* code holds corrupt indices and the fix does
not migrate it, so one asset can carry a mix of pre-fix and post-fix profiles. Worth a changelog
note or a detect-and-warn in `skeleton.audit_skin_weights`. `Mesh.Build()` runs inside the handler
(`SkeletalMeshHandler.cpp:196`, `:769`), so a wrong index is persisted immediately — test on a
fixture, never on project content.

## History
- `#1-found-by-inspection` `OPEN` reporter — Identified while implementing `skeleton.audit_skin_weights`; index-space mismatch confirmed against UE 5.8 source at the three cited locations. Runtime evidence added from integration pass 4: the project's own creep meshes are 6-section, so the precondition for corruption is met by real content in this repo.
- `#2-corrected-four-mutators-two-call-paths` `OPEN` reporter — Correction from a fresh triage at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`. (a) The ticket undercounted: it is **four** mutators, not three — `skeleton.set_vertex_weights` also routes through the shared scaffold (registered `SkeletalMeshHandler.cpp:524`, scaffold call `:576`), alongside `normalize_weights` (`:390`/`:426`) and `prune_weights` (`:449`/`:499`). (b) Acceptance-criteria-level: `skeleton.copy_weights` (`:686`) **bypasses `ApplyWeightEditToProfile` entirely** and calls `SkinWeightTransferUtils::RebuildSourceModelInfluences` inline at `SkeletalMeshHandler.cpp:761-767`, so a fix landed only in the shared helper (`SkinWeightTransferUtils.h:490`) repairs three verbs and silently misses the fourth, with no compile error and no failing scaffold-only test. Both call sites must change (`SkinWeightTransferUtils.h:490`, `SkeletalMeshHandler.cpp:766`), plus the existing test call site `TestAnimationHandlers.cpp:3242` if the helper signature grows an `FSkeletalMeshLODModel` parameter; preferred shape is to make `copy_weights` call `WriteSkinWeightProfile` outright and delete its private copy of the persist logic. (c) Recorded that this defect is stacked on a more fundamental one — all four verbs write a *named skin-weight profile*, never the LOD's base section skinning (`SkinWeightTransferUtils.h:473-491`; `Sections[*].SoftVertices` is never assigned) — filed separately as `B-weight-mutators-write-profile-not-base-skinning`, so a fix here must not be read as "the mutators now write skin weights". (d) Added the engine primitive for the fix (`FSkeletalMeshLODModel::GetSectionFromVertexIndex` + `Section.BoneMap[]`, per `MeshUtilities.cpp:5761-5762`), the `Build.cs`-unchanged finding, the `MeshUtilities.cpp:5772-5779` legacy-path inconsistency as a known red herring, and the mixed pre-fix/post-fix profile migration risk. No status change — still OPEN, still unfixed at that HEAD.
- `#3-map-through-bonemap-one-call-path` `IN-REVIEW` developer — Fixed, and the second call path was deleted rather than patched. (a) `SkinWeightTransferUtils::RebuildSourceModelInfluences` now takes `const FSkeletalMeshLODModel&` as a **required leading parameter** and maps every influence through its vertex's `FSkelMeshSection::BoneMap` before writing `FVertInfluence::BoneIndex`; it returns the count of influences dropped for want of a map entry (dropped, never defaulted to bone 0 — bone 0 is the root on every skeleton, so a fallback would fabricate an influence that passes every downstream check). The required parameter is the anti-drift guard this ticket asked for: a call site that does not pass a LOD model now **fails to compile** instead of silently keeping the old behaviour. (b) New pure helpers in the same header: `FindSectionForVertex` (deliberately NOT `FSkeletalMeshLODModel::GetSectionFromVertexIndex`, whose out-of-range guard is commented out at `SkeletalMeshLODModel.cpp:1008-1009` so it answers "last section, vertex 0" past the end), `ResolveSectionLocalBone`, `ResolveBoneToSectionLocalSlot`. (c) `skeleton.copy_weights` no longer has its own persist logic — it now calls `WriteSkinWeightProfile`, the preferred shape named at `:62-64`, which also closes `B-copy-weights-duplicates-profile-info`. (d) A further defect surfaced while fixing it and is filed as `B-copy-weights-source-section-bone-slots-untranslated`: the copied slots index the SOURCE section's BoneMap, so they need a source-local -> RefSkeleton -> target-local translation, added as `TranslateCopiedWeightsBetweenLODs`. (e) `skeleton.set_vertex_weights` now takes `boneName`/`boneIndex` in reference-skeleton space and translates to the section-local slot, with an all-or-nothing pre-flight (see `B-set-vertex-weights-boneindex-unvalidated-index-space`). (f) Regression tests, all on two-section fixtures with disjoint bone maps: `PinWright.skeleton.skin_weights.BoneIndexSpaceRoundTrip` and `PinWright.skeleton.copy_weights.TranslatesBetweenSectionBoneMaps` in the new `Source/PinWright/Private/Tests/Gameplay/TestSkinWeightBaseReadback.cpp`, plus `TestAnimationHandlers.cpp`'s `copy_weights.ClosestVertexTransfer`, whose rebuild fixture now uses a non-identity bone map `{0,10,20,30,...}` so slots 3/7/9 must come back as bones 30/70/90 — it previously asserted 3/7/9, i.e. it asserted the bug. **NOT COMPILED, NOT RUN** — the editor holds the DLL; an integration pass must build and run the suite. The stacked ticket `B-weight-mutators-write-profile-not-base-skinning` is addressed separately and this fix must still not be read as "the mutators now write skin weights".
