---
id: F-anim-set-bone-key-proper
title: "Reimplement animation.authoring.set_bone_key to splice one key at a frame (removed: it ignored `frame` and wiped the whole bone track)"
status: OPEN
severity: Medium
category: feature
tags: [animation, anim-sequence, set-bone-key, bone-track, reimplement, rpc-audit]
---

# Reimplement `animation.authoring.set_bone_key` properly (per-frame key splice)

`animation.authoring.set_bone_key` was **removed** in the batch-2 RPC audit
recorded in [`E-rpc-audit-43-record`](E-rpc-audit-43-record.md) because it was a
destructive write masquerading as a per-frame key set. The capability itself
(write a single location/rotation/scale key at a given frame into an existing
bone track, leaving every other frame intact) is the entire point of hand-keyed
bone animation and is now missing from the surface.

## What the removed version did wrong

The handler accepted a `frame` parameter, **ignored it**, and rewrote the bone
track's key array wholesale on every call. So the natural keying loop destroyed
its own work:

1. `set_bone_key {bone: pelvis, frame: 0, location: ...}` -> track holds the frame-0 pose.
2. `set_bone_key {bone: pelvis, frame: 30, location: ...}` -> track is **wiped** and holds only the new pose. Frame 0 is gone.
3. `set_bone_key {bone: pelvis, frame: 60, location: ...}` -> wiped again.

Each call returned `success:true`, and the aggregate `rawTrackCount` readback
that existed at the time still reported the track, so the loss was invisible in
band. A caller keying an N-frame pose sequence ended up with whatever the last
call wrote, applied across the track, and no error anywhere.

## What survives, and what is actually missing

This is a **write verb only** gap. The rest of the bone-track surface is intact:

- `animation.authoring.add_bone_track` still exists (it creates/adds the track).
- Per-bone-track **readback works**: the `boneTracks[]` array
  (`{boneName, keyCount}`) shipped on `describe_sequence`, `list_curves`, and
  `get_animation_info` under
  [`E-rpc-animation-bone-track-readback`](E-rpc-animation-bone-track-readback.md).

So the readback that was built to verify keyed bones now has nothing that can
write them per-frame. Reinstating a correct `set_bone_key` closes the loop.

## Proper implementation

Splice, do not replace. The controller/data-model path the removed handler
already used is the right one, it just needs to preserve the existing keys:

- Read the track's current keys off the data model
  (`IAnimationDataModel::GetBoneTrackTransforms` for the named bone, the same
  accessor the `boneTracks[]` readback uses).
- Validate `frame` against the sequence's frame count and fail loud
  (`INVALID_PARAMS`) when it is out of range, rather than clamping silently.
- Overwrite **only** the requested frame's entry in the key arrays (location /
  rotation / scale, applying only the channels the caller supplied and leaving
  the others at their current value), then write the full, preserved array back
  through `IAnimationDataController::SetBoneTrackKeys` (the same controller call
  `add_bone_track` uses).
- Echo the resulting `keyCount` so the caller can cross-check against the
  `boneTracks[]` readback in the same round-trip.

**Fix:** new handler in the animation authoring sequence handler where the
removed method lived. Regression test: create a sequence, add a bone track, then
key frames 0, 30 and 60 in three separate calls and assert **all three** keys
survive with their distinct transforms (the removed version leaves exactly one,
which is the differential proof). Add a case asserting an out-of-range `frame` is
rejected rather than silently folded onto the last frame.

## History
- `#1-reimpl-after-audit` `OPEN` reporter - Filed to reinstate the wanted capability removed by the batch-2 RPC audit ([`E-rpc-audit-43-record`](E-rpc-audit-43-record.md)). The removed `animation.authoring.set_bone_key` accepted a `frame` param, ignored it, and rewrote the whole bone-track key array on every call, so a three-key pose loop (frames 0/30/60) silently ended up with only the last key while every call reported success. The rest of the surface survives: `animation.authoring.add_bone_track` still exists and per-bone-track readback works (`boneTracks[]` on describe_sequence/list_curves/get_animation_info, shipped by [`E-rpc-animation-bone-track-readback`](E-rpc-animation-bone-track-readback.md)), so this is a per-frame WRITE verb only. Proper impl: read the existing keys via `IAnimationDataModel::GetBoneTrackTransforms`, overwrite only the requested frame's entry (honoring only the supplied channels), write the preserved array back through `IAnimationDataController::SetBoneTrackKeys`, validate `frame` against the sequence length and fail loud out of range, and echo the resulting `keyCount`.
