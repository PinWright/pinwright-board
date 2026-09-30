---
id: F-animation-check-morph-curves
title: "No check that an animation's curves will actually drive the mesh's morph targets (name match plus MorphTarget curve metadata flag)"
status: OPEN
severity: Medium
category: feature
tags: [animation, curves, morph-targets, audit, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# No curve-to-morph-target check

`animation.authoring.list_curves` and `skeleton.list_morph_targets` exist separately. The morph-target
flag on curve metadata can be written through `.pwskel`
(`Source/PinWright/Private/PwSkel/PwSkelAssetCreate.cpp:407-416`) but no verb reads it, so "will this
facial clip move the face" cannot be answered.

Name matching alone is not the answer. At runtime a curve drives a morph only if its metadata
`Type.bMorphtarget` is set on the Skeleton or on the mesh:
- Skeleton metadata → `ECurveElementFlags::MorphTarget`: `C:/UE_5.8/Engine/Source/Runtime/Engine/Private/BoneContainer.cpp:288-301`.
- Mesh `UAnimCurveMetaData` → same flag: `BoneContainer.cpp:365-376`.
- Only flagged curves reach `MorphTargetCurve`: `C:/UE_5.8/Engine/Source/Runtime/Engine/Private/Animation/AnimInstanceProxy.cpp:3597`.

Competitors: ue-mcp `compare_curves_to_morph_targets` matches names only with a `TSet` and ignores the
flag (`AnimationHandlers_Sequence.cpp:2103-2170`), so it passes curves that will never move a morph.
VibeUE reads/writes the flag with no mesh comparison.

## Engine API

`USkeleton::GetCurveMetaData` `Skeleton.h:387`, `GetCurveMetaDataMorphTarget` `:449`; mesh
`GetAssetUserData<UAnimCurveMetaData>()` (`AnimCurveMetadata.h:199`); `USkeletalMesh::FindMorphTarget`
`SkeletalMesh.h:2757`; `IAnimationDataModel::GetFloatCurves` `IAnimationDataModel.h:265`. FName-keyed
curve metadata exists across 5.3-5.8.

## Proposed scope

Read-only audit `animation.check_morph_curves` on the audit framework (`docs/rpc-design.md` §18):
- `assetPaths[]` (AnimSequence, AnimMontage or PoseAsset; required), `skeletalMeshPath` (required — the
  answer depends on the mesh).
- Per curve: `{curve, morphOnMesh, flagSource: skeleton|mesh|none, maxLod, drives}`; plus
  `morphsWithoutCurve[]` and a pass/fail verdict derived through `FVerdict::DerivePass`.
- Leaves dirty flags unchanged.

## Acceptance

- A fixture curve whose name matches a morph but carries no flag reports `drives:false`, `flagSource:none`.
- A flagged, matching curve reports `drives:true`.
- `morphsWithoutCurve` lists unanimated morphs.
- No package dirty-flag change after the call.

**Effort:** S.

## History
- `#1-no-morph-curve-check` `OPEN` reporter — Filed from the 2026-09-30 animation gap analysis at plugin HEAD `2580e7f4`. Severity Medium: the correct answer needs the metadata flag, which no verb reads.
