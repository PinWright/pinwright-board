---
id: B-animation-setup-retargeting-does-not-retarget
title: "animation.setup_retargeting reports assets as retargeted after only copying them and changing their Skeleton pointer"
status: IN-REVIEW
severity: High
category: bug
tags: [animation, retargeting, skeleton, false-success, wrong-result, verification]
---

# `setup_retargeting` never runs a retarget operation

The RPC promises to batch-retarget animation sequences between Skeletons (`Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/AnimationHandler.cpp:974-983`). Its successful path only duplicates the source, loads the copy, calls `DestinationSequence->SetSkeleton(TargetSkeleton)`, passes it to the mark-dirty-only `McpSafeAssetSave`, and appends it to `RetargetedAssets` (`:1033-1104`); its own log admits that retargeting still requires IK Rig setup (`:1099-1101`). It then sends success with `retargetedAssets` (`:1107-1137`).

The engine call does not remap motion: `UAnimationAsset::SetSkeleton` invokes `OnSetSkeleton`, assigns the pointer and GUID (`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/AnimationAsset.cpp:384-397`), while `UAnimSequence::OnSetSkeleton` only waits for compression (`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/Animation/AnimSequence.cpp:3749-3752`). Bone tracks remain authored for the source skeleton. The caller receives a successful “retarget” even when the target hierarchy needs actual IK mapping. `McpSafeAssetSave` itself only marks dirty/registers (`Plugins/PinWright/Source/PinWright/Private/Utils/AssetUtils.cpp:503-518`), so the response also provides no disk-persistence result.

Route the verb through a real IK Retargeter export/batch operation, using the existing IK Rig/IK Retargeter authoring surface, and verify the produced sequence against the target hierarchy before reporting it. If the required retarget setup is absent, reject with a typed unsupported/setup-required error; never label a pointer swap as retargeting.

**Workaround:** use `animation.authoring.create_ik_retargeter`, configure chains, and run an actual engine IK-retarget export outside this verb.

## Related

- Catalog: `terminal-success-before-completion-or-invariant`, `request-echo-not-result-readback`, `incomplete-validator-false-green`

## Fix

**Verdict: TRUE.** UE 5.8's real `UIKRetargetBatchOperation::RunBatchRetarget` requires a source SkeletalMesh, target SkeletalMesh, and configured `UIKRetargeter`, while this RPC accepts only Skeletons. `UAnimationAsset::SetSkeleton` still only assigns the Skeleton/GUID, so the feasible fix within the existing signature is an honest compatibility-copy contract: the result now reports `retargeted: false`, `duplicatedWithSkeletonSwap`, `duplicatedAssets`, and truthful save fields, and the registry description and wiki say that tracks are not remapped.

Files: `Source/PinWright/Private/Handlers/Animation/AnimationHandler.cpp`, `Source/PinWright/Private/Tests/Animation/TestSetupRetargetingContract.cpp`, and `Docs/wiki-src/animation.md`. Regression ID: `PinWright.animation.setup_retargeting.OverwriteStagesSourceBeforeReplacement`.

Deliberately unchanged: no new source-mesh, target-mesh, or IK Retargeter parameter was added, and no IK batch export is attempted. The method name and legacy `_Retargeted` suffix remain for compatibility, but no response field claims that the tracks were retargeted.

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed the RPC reaches only duplicate + SetSkeleton + mark-dirty, and engine SetSkeleton does not retarget tracks; no editor, build, or test was run.
- `#2-honest-skeleton-swap-contract` `IN-REVIEW` developer — Replaced the false retarget-success contract with explicit duplicate-plus-Skeleton-swap fields, documented the required IK batch inputs from UE 5.8 source, and added structural response coverage.
- `#3-verifier-hardening` `IN-REVIEW` developer — Replaced source-text assertions with a handler-harness test that verifies `retargeted:false`, `duplicatedWithSkeletonSwap:true`, the source-derived data GUID, target Skeleton, and durable output file.
