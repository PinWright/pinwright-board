---
id: B-blend-space-add-sample-silent-drop
title: "animation.authoring.add_aim_offset_sample / add_blend_sample report success while silently dropping rejected samples — the int32 return of UBlendSpace::AddSample is discarded"
status: IN-REVIEW
severity: High
category: bug
tags: [animation, blend-space, aim-offset, add_aim_offset_sample, add_blend_sample, sample-add-silent-drop, silent-false-success, unchecked-addsample-return]
encounters: 1
lastSeen: 2026-07-13T10:40:43.4586441+03:00
claimedBy: fuzz2
claimedAt: 2026-07-13T10:56:46.4547864+03:00
---

# `add_aim_offset_sample` / `add_blend_sample` report success but silently drop rejected samples

Both blend-space sample-add verbs call `UBlendSpace::AddSample(UAnimSequence*, const FVector&)` and **throw away its return value**, then unconditionally report success. `AddSample` returns an `int32` — the new sample index, or `INDEX_NONE` when the animation/coordinate is rejected — so when the engine rejects the sample the handler still tells the caller "sample added" while the asset gains nothing. The caller builds what it believes is a populated blend/aim grid and gets an empty one.

The most acute and always-reproducing case is **`add_aim_offset_sample` with any non-additive clip**: `UAimOffsetBlendSpace::IsValidAdditiveType` accepts **only** `AAT_RotationOffsetMeshSpace` (rotation-offset additive), so an ordinary locomotion clip (`AAT_None`) is rejected 100% of the time. Every mannequin/ThirdPerson clip a caller naturally reaches for (Jog_Fwd, ThirdPersonRun, Walk_Fwd, Wave_Loop) is `AAT_None`, so `add_aim_offset_sample` reports success on each while adding nothing, leaving `samples: []`.

## What's wrong

`animation.authoring.add_aim_offset_sample` (`AnimationAuthoringHandler_BlendSpace.cpp:730` for the AimOffset branch, `:709` for the plain-BlendSpace fallback branch) discards `AddSample`'s return:

```cpp
// Add sample with yaw/pitch coordinates
FVector SampleValue(Yaw, Pitch, 0.0f);
AimOffset->AddSample(Animation, SampleValue);          // line 730 — int32 return discarded

AnimationAuthoringHelpers::SaveAnimAsset(AimOffset, bSave);

Ctx.SendSuccess(TEXT("Aim offset sample added"));       // line 734 — always "success"
```

`animation.authoring.add_blend_sample` shares the identical pattern (`AnimationAuthoringHandler_BlendSpace.cpp:500`):

```cpp
// Add sample
BlendSpace->AddSample(Animation, SampleValue);          // line 500 — int32 return discarded
AnimationAuthoringHelpers::SaveAnimAsset(BlendSpace, bSave);
Result->SetBoolField(TEXT("success"), true);            // line 505 — always true
Result->SetStringField(TEXT("message"), TEXT("Blend sample added"));  // line 506
```

Engine ground truth (`C:\UE_5.7\Engine\Source\Runtime\Engine\Private\Animation\BlendSpace.cpp:1647`):

```cpp
int32 UBlendSpace::AddSample(UAnimSequence* AnimationSequence, const FVector& SampleValue)
{
    ExpandRangeForSample(SampleValue);
    const bool bValidSampleData = ValidateSampleValue(SampleValue) && ValidateAnimationSequence(AnimationSequence);
    if (bValidSampleData)
    {
        SampleData.Add(FBlendSample(AnimationSequence, SampleValue, true, bValidSampleData));
        UpdatePreviewBasePose();
    }
    return bValidSampleData ? SampleData.Num() - 1 : INDEX_NONE;   // INDEX_NONE == nothing added
}
```

And the aim-offset additive gate (`C:\UE_5.7\Engine\Source\Runtime\Engine\Private\Animation\AimOffsetBlendSpace.cpp:16`):

```cpp
bool UAimOffsetBlendSpace::IsValidAdditiveType(EAdditiveAnimationType AdditiveType) const
{
    return (AdditiveType == AAT_RotationOffsetMeshSpace);
}
```

`ValidateAnimationSequence` -> `IsAnimationCompatible` -> `IsValidAdditiveType(AAT_None)` returns false for an AimOffset, so `AddSample` returns `INDEX_NONE` and adds nothing — but the handler reports success anyway.

`add_blend_sample` is enumerated via the shared-pattern probe (not independently reproduced this iteration): a plain `UBlendSpace` accepts `AAT_None`, so it works for the common in-range case, but it discards the same `int32` return and will therefore report the same false success whenever `AddSample` rejects the input — an out-of-range sample coordinate or a skeleton-incompatible clip. Same defect, narrower trigger.

## What it should do

Capture `AddSample`'s return and treat `INDEX_NONE` as failure: send an error (e.g. `SAMPLE_REJECTED`) whose message names the likely cause — for an aim offset, that the clip must be a rotation-offset-mesh-space additive animation, not `AAT_None`; for a blend space, that the coordinate is out of the axis range or the clip's skeleton/additive type is incompatible with existing samples. Never report "sample added" when the sample count did not change. Optionally read back `GetNumberOfBlendSamples()` before/after as a belt-and-suspenders check.

## Verbatim repro (live editor, this host)

1. `animation.authoring.create_aim_offset {name:"AO_Judge_Repro", path:"/Game/Characters/Mannequin_UE4/Animations", skeletonPath:"/Game/Characters/Mannequin_UE4/Meshes/SK_Mannequin_Skeleton"}` -> `{"assetPath":".../AO_Judge_Repro","success":true,"message":"Aim Offset 'AO_Judge_Repro' created"}` (create genuinely works; className AimOffsetBlendSpace).
2. `animation.authoring.add_aim_offset_sample {assetPath:".../AO_Judge_Repro", animationPath:"/Game/Characters/Mannequin_UE4/Animations/Jog_Fwd", yaw:0, pitch:0}` -> `{"message":"Aim offset sample added"}` (note: reports success; the coordinate (0,0) is in-range, so the ONLY rejection cause is the non-additive clip).
3. `asset.dump {assetPath:".../AO_Judge_Repro"}` -> `blend_space.json` reads back `"samples": []` — the sample was NOT added despite the success message.
4. Ground truth on the clip: `Jog_Fwd/anim_sequence.json` reads `"additiveType": "AAT_None"` — not the `AAT_RotationOffsetMeshSpace` an aim offset requires.

severity rationale: impact=silent-false-success (a normal add-sample call reports "added" while the asset stays empty; the caller trusts the lie and ships an empty aim/blend grid) × reach=common animation-authoring path (aim-offset/blend-space sample blocking, and `add_aim_offset_sample` fails on EVERY non-additive clip a caller would naturally use) -> High.

## History
- `#1-initial-repro` `OPEN` reporter — SEED-mode iteration seeded on `animation.authoring.create_aim_offset` (which works correctly). Attempt task: build an Aim Offset for the UE4 mannequin and block out the aim grid with existing mannequin clips as placeholder poses. REPLAY-CONFIRMED on the live editor: `add_aim_offset_sample` returned `{"message":"Aim offset sample added"}` for `Jog_Fwd` at in-range (0,0), yet `asset.dump` blend_space.json came back `samples: []`. Source-confirmed the discarded `int32` return in both `add_aim_offset_sample` (`AnimationAuthoringHandler_BlendSpace.cpp:730`,`:709`) and sibling `add_blend_sample` (`:500`), and the engine reject path (`BlendSpace.cpp:1647` returns `INDEX_NONE`; `AimOffsetBlendSpace.cpp:16` `IsValidAdditiveType` accepts only `AAT_RotationOffsetMeshSpace`). Distinct from `B-create-blend-space-axis-config-dropped-on-57` (that ticket owns the `set_axis_settings`/create-side axis-config no-op — a different root cause; the same attempt also re-hit that `set_axis_settings` no-op on an AimOffset, already covered there and not re-filed here). Filed as a family ticket: `add_aim_offset_sample` reproduced, `add_blend_sample` enumerated via the shared-pattern probe. Outcome tool_bug; culprit `animation.authoring.add_aim_offset_sample`.
- `#2-lead-decision` `IN-REVIEW` developer — GO (valid; red test reproduced the defect on the AimOffset branch). Verified the discarded `UBlendSpace::AddSample` int32 return in current source at `add_blend_sample` (`AnimationAuthoringHandler_BlendSpace.cpp:500`) and `add_aim_offset_sample` (`:709` fallback + `:730` AimOffset), and the engine reject paths (`BlendSpace.cpp:1660` returns INDEX_NONE; `AimOffsetBlendSpace.cpp:18` `IsValidAdditiveType` accepts only `AAT_RotationOffsetMeshSpace`; base `BlendSpace.cpp:1114` accepts `AAT_None`). Not a duplicate of `B-create-blend-space-axis-config-dropped-on-57` (that owns `set_axis_settings`/create axis-config — a different method/file). Severity High stands (silent false-success on a common authoring path). Decision: capture the return and, on INDEX_NONE, `SendError` a new `SAMPLE_REJECTED` code (registered in `ErrorCodes.h`) naming the additive/skeleton/duplicate cause, across the full ticket scope (both verbs / all three sites) via one shared helper; adopt the existing red test and add an `add_blend_sample` duplicate-coordinate reject test. (The ticket's `add_blend_sample` "out-of-range coordinate" repro is inaccurate — `ExpandRangeForSample` widens the axis first — so the reject test uses a duplicate coordinate instead; no reword since defect/fix/severity are correct.)
- `#3-lead-fix-shipped` `IN-REVIEW` developer — Implemented as decided. Added a shared file-local helper `PinWrightBlendSpaceSampleAdd::AddSampleOrReportRejection` that captures `UBlendSpace::AddSample`'s int32 return and, on INDEX_NONE, sends `SAMPLE_REJECTED` (new code registered in `Handlers/ErrorCodes.h`) with an additive/skeleton/duplicate hint; wired into all three sites (`add_blend_sample` :500, `add_aim_offset_sample` fallback :709 + AimOffset :730). Files: `Source/PinWright/Private/Handlers/Animation/AnimationAuthoringHandler_BlendSpace.cpp`, `Source/PinWright/Private/Handlers/ErrorCodes.h`, `Source/PinWright/Private/Tests/Gameplay/TestAimOffsetAddSampleSilentDrop.cpp`. Tests: adopted + strengthened the red test `PinWright.animation.authoring.add_aim_offset_sample.RejectedSampleReportsFailure` (now also asserts the SAMPLE_REJECTED code) and added `PinWright.animation.authoring.add_blend_sample.RejectedSampleReportsFailure` (duplicate-coordinate reject; also guards that the valid first add still reports success). Plugin compiled clean; both tests Result={Success}. Awaiting full-suite verification.
