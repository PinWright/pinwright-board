---
id: E-rpc-animation-extend-get-animation-info
title: "Extend `animation.authoring.get_animation_info` with additiveType / frameRateRational / rawTrackCount / skeletonAssetPath"
status: DONE
severity: Low
category: ergonomic
tags: [animation, asset-dump, parity]
---

# Extend `animation.authoring.get_animation_info` with additiveType / frameRateRational / rawTrackCount / skeletonAssetPath

`AnimSequenceDumpBuilder::BuildAnimSequenceJson` (`Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AnimSequenceDumpBuilder.cpp`) emits a richer per-sequence shape than the live introspection RPC. The dump exposes:

- `additiveType` — enum string via `StaticEnum<EAdditiveAnimationType>()` (`AAT_None` / `AAT_LocalSpaceBase` / `AAT_RotationOffsetMeshSpace`).
- `frameRateRational` — `{ numerator, denominator }` struct via `MovieSceneJsonUtils::MakeFrameRateObject(Sequence->GetSamplingFrameRate())`, while legacy numeric `frameRate` remains unchanged for existing callers.
- `rawTrackCount` — `Sequence->GetDataModel()->GetNumBoneTracks()`.
- `skeletonAssetPath` — `Sequence->GetSkeleton()->GetPathName()` (key uses the `assetPath` suffix consistently with the rest of the dump).

Live `animation.authoring.get_animation_info` (`AnimationAuthoringHandler.cpp`) only emitted `duration`, `numFrames`, `frameRate` (decimal collapse via `AsDecimal()`), `numNotifies`, `isAdditive` (bool collapse), `hasRootMotion`, and `skeletonPath`. Callers that want the additive subtype, exact rational frame rate, raw bone-track count, or skeleton path under the dump's key shape had to read `.editor-automation/asset-dumps/`. That violates the no-exclusive-dump-fields policy.

**Fix:** Additively extend the `UAnimSequence` branch of `get_animation_info` with the four fields above. Keep existing `isAdditive`, numeric `frameRate`, and `skeletonPath` fields for backward compat; add `frameRateRational`, `additiveType`, `rawTrackCount`, and `skeletonAssetPath` alongside. Reuse `MovieSceneJsonUtils::MakeFrameRateObject` and `StaticEnum<EAdditiveAnimationType>()` so the live shape matches the dump values without replacing the legacy numeric key.

## History
- `#1-initial-repro` `OPEN` reporter — `animation.authoring.get_animation_info` (AnimationAuthoringHandler.cpp:3320) returns isAdditive (bool) / frameRate (decimal) / no track count / skeletonPath, while AnimSequenceDumpBuilder emits additiveType (enum string), frameRate{numerator,denominator}, rawTrackCount, and skeletonAssetPath. Confirmed via grep that AdditiveAnimType / GetNumBoneTracks / MakeFrameRateObject (rational) are not exposed by any other animation RPC.
- `#2-reformulated-frame-rate-key` `OPEN` reporter — The parity gap is real, but replacing numeric `frameRate` with an object would break existing callers. Reformulated the requested rational field to `frameRateRational` while preserving legacy numeric `frameRate`.
- `#3-extended-animation-info` `IN-REVIEW` developer — Extended the UAnimSequence branch in AnimationAuthoringHandler.cpp with frameRateRational, additiveType, rawTrackCount, and skeletonAssetPath while preserving numeric frameRate, and added FAuthoringGetAnimationInfoAnimSequenceParityFieldsTest coverage for the compatibility shape.
- `#4-verify-fix` `DONE` tester — Verified: called animation.authoring.get_animation_info on /Game/ScifiJungle/Demo/Characters/Mannequins/Animations/Shared/A_Steering_Straight; response contains additiveType="AAT_None", frameRateRational={numerator:24,denominator:1}, rawTrackCount=164, skeletonAssetPath="/Game/.../SK_Mannequin.SK_Mannequin"; legacy fields isAdditive=false, frameRate=24, skeletonPath preserved.
