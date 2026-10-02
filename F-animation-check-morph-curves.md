---
id: F-animation-check-morph-curves
title: "No check that an animation's curves will actually drive the mesh's morph targets (name match plus MorphTarget curve metadata flag)"
status: DONE
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
- `#2-check-morph-curves-verb` `IN-REVIEW` developer — Added read-only audit `animation.check_morph_curves` (`assetPaths[]` AnimSequence/AnimMontage/PoseAsset, `skeletalMeshPath`, `failOn`) in `Source/PinWright/Private/Handlers/Animation/MorphCurveAuditHandler.cpp`. Replays `CacheRequiredAnimCurves` precedence: a mesh `UAnimCurveMetaData` entry carrying any flag replaces the Skeleton's, else the mesh Skeleton's `Type.bMorphtarget`; `drives = flag && FindMorphTarget-name match`. Per asset: `curves[] {curve, morphOnMesh, flagSource, maxLod, drives}`, `morphsWithoutCurve[]` (morphs no curve in that asset drives), `status flagged|clean|unrunnable`; findings `unflagged_morph_curve` (error) and `flagged_curve_without_morph` (warning); verdict via `FVerdict::DerivePass`, `PassRuleText(false)`. Malformed path -> `INVALID_ARGUMENT` before the sweep; missing/wrong-type asset -> `unrunnable` (`ASSET_NOT_FOUND` / `INVALID_ASSET_TYPE`). Montage curves include each segment sequence's curves. Wiki: `docs/wiki-src/animation.md` (section + H3). Test `PinWright.animation.check_morph_curves.FlagAndNameDecideDrives` (SkeletalCube copy, 4 one-delta morphs, skeleton + mesh metadata): asserts the four flag/name combos, `morphsWithoutCurve == [Blink, Smile]`, 1 error / 1 warning, pass false under failOn error, true under none, false again with a missing asset (unrunnable), and unchanged dirty flags on mesh/skeleton/clip packages. Compile-checked with -SingleFile; not run yet.
- `#3-fixround-param-type-and-fixture` `IN-REVIEW` developer — Suite round fixes. (1) `assetPaths` declared `path|array` (was `array`), so the dispatcher's path-separator rule reaches its elements; clears `PinWright.infra.contract.PathParamTypes.PathShapedParamsDeclareAPathType` (`get_live_pose` has no path-shaped params). (2) Test fixture: `RegisterMorphTarget(Morph)` with render-data invalidation runs `FScopedSkeletalMeshPostEditChange`, whose editor rebuild regenerates morphs from the SkeletalCube copy's imported model (none) and dropped all four fixture morphs; the test now registers with `bInvalidateRenderData=false` and calls `InitMorphTargets()` to rebuild the name map. Verb code unchanged apart from the declaration. Compile-checked with -SingleFile.
- `#4-fixround2-read-data-model` `IN-REVIEW` developer — Verb bug, not fixture: curve collection read `UAnimSequenceBase::GetCurveData()` (the deprecated runtime `RawCurveData` copy, synced from the model only on some notifications). In suite run w23-full2 it came back EMPTY for a sequence whose data model held four float curves, so every curve read as absent (`morphsWithoutCurve` listed all four morphs, 0 findings, pass:true — a false green a real caller with a freshly authored clip would get). Now reads `GetDataModel()->GetFloatCurves()` when `IsDataModelValid()` (the editor authority, same source `animation.authoring.get_curve_keys` uses), falling back to `GetCurveData()` otherwise. File: `Handlers/Animation/MorphCurveAuditHandler.cpp`. Compile-checked with -SingleFile. Note: `animation.authoring.list_curves` / `animation.describe_sequence` (`AnimSequenceDumpBuilder::BuildCurvesArrayJson`) still read `GetCurveData()` and may share the staleness; not verified.
- `#5-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.animation.check_morph_curves.FlagAndNameDecideDrives` passed in w23-final (no skip): a name-matching unflagged curve -> `drives:false`, `flagSource:none`; a flagged matching curve -> `drives:true`; `morphsWithoutCurve == [Blink, Smile]`; dirty flags on the mesh, skeleton and clip packages unchanged; plus the finding and verdict checks. All four Acceptance bullets are met after #4's switch to the data model. `PinWright.infra.contract.PathParamTypes.PathShapedParamsDeclareAPathType` also passed. Open side note from #4: `list_curves` / `describe_sequence` still read `GetCurveData()` (tracked by `B-anim-curves-read-runtime-copy`).
