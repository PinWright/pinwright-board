---
id: F-anim-composite-asset-creator
title: "No `UAnimComposite` create/segment handlers"
status: DONE
severity: Low
category: feature
tags: [animation, anim-composite, asset-creation, anim-track, anim-segment]
---

# No `UAnimComposite` create/segment handlers

The animation authoring surface covers five sub-workflows — sequences,
montages, anim BPs, blend spaces, control rigs — but omits
`UAnimComposite`. A composite is a chain-of-sequences asset:
`UAnimComposite : UAnimCompositeBase`, whose `AnimationTrack`
(`FAnimTrack`) holds an array of `FAnimSegment` entries pointing at
`UAnimSequenceBase` assets with their own start position and play rate.
Distinct from `UAnimMontage`, which has slots, sections, branching
points, and notify tracks — composites are the lighter "stitch these
sequences end-to-end" container.

Zero hits for `UAnimComposite`, `AnimSegment`, or `composite` (in the
asset-class sense) across `docs/rpc-method-reference.generated.md` and
`Source/EditorAutomationRpcGateway/Private/Handlers/Animation/`. The
existing `CompositeSections` references in `AnimationAuthoringHandler.cpp`
are `FCompositeSection` on `UAnimMontage` — unrelated.

**Use cases blocked:**

1. Programmatic stitch-up of mocap takes into a single playable
   composite for previews or test fixtures.
2. Round-tripping a composite asset through dump → re-create (the
   segment chain is opaque in `properties.json`).
3. Authoring training/idle loops by chaining short sequences without
   the overhead of montage slots/sections.

**Workaround:** Create the asset manually in the editor, then use
generic `asset.*` and `set_property` calls to poke at
`AnimationTrack.AnimSegments` — but the `FAnimSegment` struct has
~10 fields (StartPos, AnimStartTime, AnimEndTime, AnimPlayRate,
LoopingCount, etc.) and no segment-add helper, so this is impractical
at scale.

**Proposal:** Two handlers, mirroring the existing montage-section
pattern in `AnimationAuthoringHandler.cpp`:

1. `animation.authoring.create_composite(name, skeletonPath, path?,
   save?)` — create a new `UAnimComposite` asset bound to the supplied
   `USkeleton`. Mirror the existing `create_montage` /
   `create_animation_sequence` handler shape (factory + package +
   `SaveAnimAsset()`).

2. `animation.authoring.add_composite_segment(assetPath, animationPath,
   startPos?, animPlayRate?, animStartTime?, animEndTime?,
   loopingCount?, save?)` — append an `FAnimSegment` to
   `UAnimComposite::AnimationTrack.AnimSegments`. Required fields:
   target composite asset path + source `UAnimSequenceBase` asset path.
   Optional fields default to append-at-end placement, the segment's
   natural range, 1x play rate, and one loop. Returns the new segment
   index and segment count.

A segment remover / reorder helper can wait until there are 2+
concrete callers (YAGNI).

**Implementation notes:**

- Engine entry points: `UAnimComposite::AnimationTrack` (an
  `FAnimTrack`) → `AnimSegments` (a `TArray<FAnimSegment>`).
  `FAnimSegment::AnimReference` is a `UAnimSequenceBase*`. The track
  has `SortAnimSegments()` to keep the array ordered by `StartPos`
  after insertion.
- Asset creation: `UAnimCompositeFactory` lives in
  `UnrealEd`/`AnimationEditor` module — already in scope for the
  existing `create_montage` path; no new module dependency.
- Saving: reuse current `SaveAnimAsset()` behavior.
- Validation: segment's `AnimReference` skeleton must match the
  composite's skeleton via `IsCompatibleForEditor` — surface as
  `SKELETON_MISMATCH` error to match the existing montage error
  vocabulary.

## History
- `#1-initial-repro` `OPEN` reporter — Wiki and handler source both omit `UAnimComposite` from the animation authoring surface (no hits for `UAnimComposite`/`AnimSegment`/`create_composite` in `rpc-method-reference.generated.md` or `Handlers/Animation/`; existing `CompositeSections` references are montage `FCompositeSection`, unrelated). Proposes `create_composite(assetPath, skeleton)` plus `add_composite_segment(assetPath, sequencePath, startPos, animPlayRate?)`, modelled on the existing `create_montage` + montage-section handlers in `AnimationAuthoringHandler.cpp`. Engine entry point: `UAnimComposite::AnimationTrack` (`FAnimTrack` → `TArray<FAnimSegment>`). Trivial extension; defer segment-remove/reorder until 2+ callers.
- `#2-corrected-handler-contract` `OPEN` developer — Corrected the proposed handler contract to match existing animation-authoring params (`name`/`path`/`skeletonPath`, `animationPath`), current `SaveAnimAsset()` behavior, and UE editor skeleton compatibility validation.
- `#3-add-composite-handlers` `IN-REVIEW` developer — Added `animation.authoring.create_composite` and `animation.authoring.add_composite_segment` in `AnimationAuthoringHandler.cpp`, documented the composite workflow, and added `FAuthoringCompositeCreateAndAddSegmentTest` coverage.
- `#4-verify-composite-handlers` `DONE` tester — Verified: `animation.authoring.create_composite` created `/Game/App/UI/Test/AC_McpVerifyTemp_F_anim_composite_asset_creator` as `AnimComposite`, and `animation.authoring.add_composite_segment` appended `/Game/Characters/Heroes/Mannequin/Animations/Locomotion/Rifle/MM_Rifle_Idle_ADS.MM_Rifle_Idle_ADS` with `segmentIndex: 0`, `segmentCount: 1`, `compositeLength: 3.4000000953674316`; temporary asset deleted afterward with `asset.delete`.
