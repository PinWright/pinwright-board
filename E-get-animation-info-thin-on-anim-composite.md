---
id: E-get-animation-info-thin-on-anim-composite
title: "`animation.authoring.get_animation_info` on a UAnimComposite emits only {assetType} — no skeleton / duration / segments, so the create_composite + add_composite_segment round-trip is unverifiable without asset.dump (the composite branch missing from the get_animation_info parity family)"
status: OPEN
severity: Low
category: ergonomic
tags: [animation, anim-composite, asset-dump, parity, get-animation-info]
encounters: 1
lastSeen: 2026-07-02T07:34:50.0384184+03:00
---

# `get_animation_info` on a UAnimComposite is `{assetType}`-only — skeleton / duration / segments live only in the asset.dump sidecar

`get_animation_info` is documented as the introspection dual used to confirm an
asset's shape after authoring (the *Inspect-after-mutate* loop on the
`animation.authoring` wiki overlay). Its sibling branches have all been extended
to deliver full readback parity by delegating to the matching dump builder:

- `UAnimSequence` — rich shape (`E-rpc-animation-extend-get-animation-info`, DONE).
- `UAnimMontage` — `JsonBuilders::MergeMissingFields(AnimInfo, AnimMontageDumpBuilder::BuildAnimMontageJson(Montage))` (`AnimationAuthoringHandler_Sequence.cpp:1861`), so sections / links / slots round-trip live.
- `UBlendSpace` — `JsonBuilders::MergeMissingFields(AnimInfo, BlendSpaceDumpBuilder::BuildBlendSpaceJson(BlendSpace))` (`AnimationAuthoringHandler_Sequence.cpp:1877`), so samples / axes round-trip live.
- `UAnimBlueprint` — `JsonBuilders::MergeMissingFields(AnimInfo, AnimGraphDumpBuilder::BuildAnimGraphJson(AnimBP))` (`AnimationAuthoringHandler_Sequence.cpp:1895`), so state machines / states / transitions round-trip live.

**The `UAnimComposite` type has no branch at all.** `UAnimComposite` is a
`UAnimCompositeBase` (a sibling of `UAnimMontage`, not a subclass), so it matches
none of the four `Cast<>` branches and falls through to the terminal `else`
(`AnimationAuthoringHandler_Sequence.cpp:1897-1900`):

```cpp
else
{
    AnimInfo->SetStringField(TEXT("assetType"), Asset->GetClass()->GetName());
}
```

That is the ONLY field emitted. The composite readback is therefore even
thinner than the (already-ticketed) montage / blend-space branches — those at
least return `skeletonPath` plus a count; the composite returns just
`{assetType}` with **no skeleton, no duration, and no segment list**:

```
call animation.authoring.get_animation_info {"assetPath":"/Game/ExampleContent/IKRig/Animations/Dino_AmbientCycle"}
-> {"animationInfo":{"assetType":"AnimComposite"},"success":true,"message":"Animation info retrieved"}
```

By contrast the AnimSequence branch on a source clip returns the full shape:

```
call animation.authoring.get_animation_info {"assetPath":"/Game/ExampleContent/IKRig/Animations/Dino_Idle"}
-> {"animationInfo":{"assetType":"AnimSequence","skeletonPath":"…/SK_DinoDragon_Skeleton","duration":0.9666…,"numFrames":30,"frameRate":30,…}, …}
```

So after the canonical composite build-out (`create_composite` →
`add_composite_segment` ×N — both DONE-shipped by `F-anim-composite-asset-creator`)
there is **no way to confirm which clips were stitched, in what order, at what
start position, or the total play length** from `get_animation_info`. The only
way to verify the Idle→Walk→Idle segment chain is to fall back to the read-only
`asset.dump` (the attempt read the composite's `properties.json`, whose
`AnimationTrack.AnimSegments[]` carry `AnimReference` / `StartPos` /
`AnimPlayRate` / loop counts and whose `SequenceLength` = the summed effective
durations). The segment list, start positions, and total length are therefore
**exclusive to the dump** — unreachable through the live introspection RPC. The
call is not wrong (assetType is accurate, no crash, valid JSON) — it is too thin
to verify the very structure `add_composite_segment` authors.

This is the same no-exclusive-dump-fields parity class as DONE
`E-rpc-animation-extend-get-animation-info` (AnimSequence) and IN-REVIEW
`E-get-animation-info-thin-on-montage` / `E-get-animation-info-thin-on-blend-space`
/ `E-get-animation-info-thin-on-anim-blueprint` (Montage / BlendSpace /
AnimBlueprint) — here for the `UAnimComposite` type, the one authorable anim
asset kind the parity family never covered (the AnimBlueprint ticket called
itself "the last branch," but it enumerated only the four existing `Cast<>`
branches and missed that composites hit the generic `else`).

**Workaround:** read the `asset.dump` `properties.json` for the composite
(`AnimationTrack.AnimSegments[]` — `AnimReference`, `StartPos`, `AnimPlayRate`,
`LoopingCount` — and top-level `SequenceLength`) for the stitched-segment chain
and total length.

**Fix:** Add a `UAnimComposite` branch to `get_animation_info` (placed so it is
tested before / independently of the `UAnimMontage` branch, since both derive
from `UAnimCompositeBase`) that emits `skeletonPath`, `duration`
(`Composite->GetPlayLength()`), and a `segments[]` array — one entry per
`Composite->AnimationTrack.AnimSegments[i]` with the referenced animation path
(`AnimSegment.GetAnimReference()->GetPathName()`), `StartPos`, `AnimEndTime`/
`AnimPlayRate`, and `LoopingCount`. The handler already iterates exactly these
accessors in `add_composite_segment` (`AnimationAuthoringHandler_Sequence.cpp`
around :1768 for `GetAnimReference()` and :1784 for `GetPlayLength()`), so the
readback reuses proven accessors. Ideally delegate to a shared composite dump
builder (mirroring the Montage/BlendSpace/AnimBlueprint branches) so the live
shape matches the sidecar. Keep `assetType` for backward compatibility.

## History
- `#1-initial-repro` `OPEN` reporter — SEED mode on `animation.authoring.add_composite_segment`; the seed verb itself worked (create_composite + 3× add_composite_segment built `/Game/ExampleContent/IKRig/Animations/Dino_AmbientCycle` = Idle→Walk→Idle, and asset.dump properties.json confirmed 3 contiguous segments with StartPos 0 / 0.96667 / 1.93333 and SequenceLength 3.86665). Culprit is `get_animation_info`, a neighbor of the seed. Replayed at HEAD: `get_animation_info {"assetPath":"/Game/ExampleContent/IKRig/Animations/Dino_AmbientCycle"}` -> verbatim `{"animationInfo":{"assetType":"AnimComposite"},"success":true,"message":"Animation info retrieved"}` — only `assetType`, no skeleton / duration / segments; the same call on the source clip `Dino_Idle` returns the rich AnimSequence shape (skeletonPath, duration 0.9666…, numFrames 30, frameRate 30, …). Source-confirmed: `AnimationAuthoringHandler_Sequence.cpp:1812/1842/1863/1879` branch only on `UAnimSequence`/`UAnimMontage`/`UBlendSpace`/`UAnimBlueprint`; `UAnimComposite` (a `UAnimCompositeBase`, not a `UAnimMontage`) matches none and falls into the terminal `else` at `:1897-1900` which emits `assetType` = `Asset->GetClass()->GetName()` and nothing else. Same no-exclusive-dump-fields parity class as DONE `E-rpc-animation-extend-get-animation-info` (AnimSequence) and IN-REVIEW montage/blend-space/anim-blueprint siblings; here for the composite type the family never covered — and thinner than those (no skeletonPath/count, just assetType), because it hits the generic else rather than a dedicated branch. The documented inspect-after-mutate loop is uncompletable from `get_animation_info` alone for composites; the attempt fell back to asset.dump properties.json. severity rationale: impact=readback-omits-fields (Medium) × reach=rare (composite introspection is a rare edge path vs. sequences) -> Low — consistent with the montage/blend-space/anim-blueprint siblings.
