---
id: E-rpc-animation-bone-track-readback
title: "No live readback of per-bone-track keys authored by add_bone_track / set_bone_key — only an aggregate rawTrackCount"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [animation, anim-sequence, bone-track, set-bone-key, readback, inspect-after-mutate, docs]
---

# No live readback of per-bone-track keys authored by add_bone_track / set_bone_key — only an aggregate rawTrackCount

`animation.authoring.add_bone_track` + `animation.authoring.set_bone_key` are the
write side for hand-keyed `UAnimSequence` bone animation (location / rotation /
scale keys per bone, per frame). But there is **no live-read RPC that surfaces
those keyed transform values back** so the documented inspect-after-mutate loop
can confirm the keys landed. The three read endpoints an author reaches for after
keying bones each fall short for *bone tracks specifically*:

- `animation.authoring.get_animation_info` — returns `rawTrackCount` (a **count**:
  `2` for two keyed bones), but no per-track name, no per-track keyCount, no key
  values. You learn *how many* tracks exist, never *which bones* or *what was keyed*.
- `animation.authoring.list_curves` — returns ONLY Float / Transform **curve**
  entries (`{name, type, keyCount}`). Raw bone tracks authored by `set_bone_key`
  are NOT curves in the UE data model, so they never appear here. On this task it
  returned only the `BreatheAmount` Float curve and zero entries for the two keyed
  bone tracks — correct per the data model, but it means there is no per-bone
  `keyCount` to cross-check against the 3 location keys (pelvis) / 3 rotation keys
  (spine_1) that were just written.
- `animation.describe_sequence` (shipped under `E-dump-rpc-parity` `#4`) — returns
  `{assetKind, path, lengthSeconds, frameRate, numFrames, notifies, curves,
  syncMarkers}`. Same gap: it carries notifies / curves / sync markers but **no
  bone-track section**, so the keyed location/rotation/scale values are still
  invisible.

Net effect: an author can WRITE bone keys but cannot READ them back through any
live RPC to verify the round-trip. The only confirmation available is the
aggregate `rawTrackCount`, which proves the track exists but not that the
intended keys (frame indices + transform values) were applied. To actually see
the keyed transforms an agent would have to fall back to the `asset.dump`
sidecar / generic property walk — the same no-exclusive-live-readback class as
the already-DONE curve and get_animation_info parity tickets, here repeated for
the **raw bone-track** branch which all three current readers skip.

This is the bone-track analogue of `F-rpc-animation-list-curves` (DONE, added
curve listing) and `E-rpc-animation-extend-get-animation-info` (DONE, added the
`rawTrackCount` aggregate). Those closed the curve and count surfaces; the
per-bone-track key values remain unreadable live.

**Workaround:** trust the `success:true` from each `set_bone_key`, and use
`rawTrackCount` from `get_animation_info` as a coarse existence check; for the
actual keyed values fall back to an `asset.dump` of the sequence.

**Fix (one of):**
- Add a live `animation.authoring.list_bone_tracks` (or extend `describe_sequence`
  with a `boneTracks: [{boneName, keyCount}]` array, mirroring the `list_curves`
  `{name, keyCount}` shape) so each keyed bone surfaces with its key count — the
  minimal cross-check the inspect-after-mutate loop needs.
- Optionally a per-track key dump (`animation.authoring.get_bone_track` →
  `{boneName, keys:[{frame, location, rotation, scale}]}`) for full round-trip
  verification, reusing the DataModel accessors
  (`GetDataModel()->GetBoneTrackByName` / `GetBoneTrackKeys`).
- **Docs:** until a reader lands, `docs/wiki-src/animation.authoring.md` should
  state explicitly in the inspect-after-mutate section that bone-track keys
  authored via `set_bone_key` are **not** surfaced by `list_curves`,
  `get_animation_info` (only `rawTrackCount`), or `describe_sequence` — so authors
  don't assume keyed bones will appear as `list_curves` Transform entries (exactly
  the wrong assumption that produced this task's success-check friction).

## History
- `#1-initial-audit` `OPEN` reporter — Process/readback gap surfaced on a clean-outcome task (focus `animation.authoring.set_bone_key`): authored A_DinoDragon_IdleBreathe with two keyed bone tracks (pelvis location 0→4→0, spine_1 rotation 0→3→0 at frames 0/30/60) plus a BreatheAmount Float curve, then ran the full readback trio to verify. Friction note: "list_curves returns ONLY the Float BreatheAmount curve and NO Transform entry for the keyed bones; the two bone tracks land as raw tracks (rawTrackCount=2, confirmed by get_animation_info and describe_sequence)... the success check assumed bone keys would surface as list_curves Transform entries, so there is no per-bone keyCount visible from list_curves to cross-check the 3 location/rotation keys." Confirmed against the board: `list_curves` (F-rpc-animation-list-curves DONE) is curves-only by design; `get_animation_info` (E-rpc-animation-extend-get-animation-info DONE) exposes `rawTrackCount` aggregate only; `describe_sequence` (E-dump-rpc-parity #4 DONE) returns notifies/curves/syncMarkers but no bone-track section. So no live RPC reads back per-bone-track keys. Same no-exclusive-live-readback class as those DONE tickets, here for the raw bone-track branch. Proposed: `list_bone_tracks` / extend `describe_sequence` with `boneTracks:[{boneName,keyCount}]` (minimal cross-check), optional per-track key dump, and a docs note on `animation.authoring.md` that bone-track keys are not live-readable today.
- `#2-retriage` `OPEN` triage — Low→Medium: no live RPC reads back authored bone-track keys, a readback omission forcing an asset.dump fallback to verify writes on a niche anim path.
- `#3-fix` `IN-REVIEW` developer — Added per-bone-track readback so `set_bone_key` / `add_bone_track` writes are verifiable through a live RPC. New `AnimSequenceDumpBuilder::BuildBoneTracksArrayJson` enumerates raw bone tracks via the non-deprecated UE 5.7 `IAnimationDataModel::GetBoneTrackNames` + `GetBoneTrackTransforms` (NOT the ticket's optional bullet's deprecated `GetBoneTrackByName` / nonexistent `GetBoneTrackKeys`) and emits sorted `{boneName, keyCount}` entries (keyCount = keyed-frame count per track); it reuses the same `<5.6` sequencer-data-model-missing-MovieScene guard as the existing `rawTrackCount` path to stay crash-free on un-initialized sequences. Wired into `BuildAnimSequenceJson` so `describe_sequence` and the `anim_sequence.json` sidecar carry `boneTracks[]`, and added it directly to `list_curves` (raw bone tracks are not curves, so they never appeared in `curves[]`) and the `get_animation_info` AnimSequence branch (alongside the aggregate `rawTrackCount`). Bumped the `anim_sequence.json` aspect version 2→3 (new serialized field). Files: `Source/PinWright/Private/Handlers/Asset/AnimSequenceDumpBuilder.{h,cpp}`, `Source/PinWright/Private/Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp`, `Source/PinWright/Private/Handlers/Asset/AssetDumpCache.cpp`, plus the `docs/wiki-src/animation.authoring.md` inspect-after-mutate doc (now states bone-track keys ARE live-readable via `boneTracks`). Regression test `PinWright.Assets.AnimSequence.DumpBuilder.BoneTracksReadback` (`Tests/Assets/TestAnimSequenceDumpBuilder.cpp`) authors two bone tracks through the production `IAnimationDataController` path (the same `AddBoneCurve` + `SetBoneTrackKeys` calls `set_bone_key` uses) and asserts both surface in `BuildBoneTracksArrayJson` and `BuildAnimSequenceJson`'s `boneTracks[]` with correct `boneName`/`keyCount` — it would fail (empty/absent `boneTracks`) if the readback were reverted.
