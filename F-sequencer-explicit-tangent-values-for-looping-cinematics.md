---
id: F-sequencer-explicit-tangent-values-for-looping-cinematics
title: "No way to set tangent VALUES (only tangentMode), so a seamless looping camera move needs decoy keys near both endpoints — RCTM_Auto forces LeaveTangent=0 on the first key and ArriveTangent=0 on the last, which makes every RPC-authored loop stop dead at the seam"
status: OPEN
severity: Medium
category: feature
tags: [sequencer, keyframes, tangents, curves, looping, cinematics, camera-path, workaround-required]
encounters: 1
lastSeen: 2026-08-27T19:40:00+05:00
---

# A looping camera move cannot be closed properly through this API

`sequence.add_keyframe` / `sequencer.add_keyframe` expose `interp` (constant | linear | cubic) and
`tangentMode` (auto | user | break | none). What they do not expose is the **tangent values**
themselves — there is no `arriveTangent` / `leaveTangent`. `tangentMode: "user"` selects the mode
that uses caller-supplied tangents while giving the caller no way to supply any.

That is a hole with one specific, common, and highly visible consequence.

## Why it bites exactly at a loop seam

`FRichCurve::AutoSetTangents` special-cases the endpoints: with `RCTM_Auto`, the **first** key's
`LeaveTangent` is forced to `0` and the **last** key's `ArriveTangent` is forced to `0`. For an
ordinary shot that is desirable — the move eases in and eases out. For a **looping** sequence, where
frame N is the same pose as frame 0 and the two are cut together, it means the camera decelerates to
a full stop at the seam and accelerates away from rest. In a 20 s loop that is a visible hitch on
every repeat, at the one frame the viewer sees most often.

Velocity at the seam is the thing an author needs to control, and it is precisely the thing the API
hands to `RCTM_Auto`.

## The workaround, and why it should not be necessary

On `/Game/Atlantis/Cine/LS_Atlantis_Flythrough` (60 fps, frames 0-1200, camera pose at 1200
identical to frame 0) I closed the loop by authoring two **decoy keys** whose only job is to shape
the tangent the endpoint is not allowed to have:

- a key at frame **40**, placed on the desired departure velocity line from frame 0;
- a key at frame **1160**, placed on the same velocity line arriving at frame 1200;

both computed from the intended seam velocity `v` as `P(0) + 40·v` and `P(1200) − 40·v`, with the
rotation channels treated the same way. The forced-zero endpoint behaviour is then confined to a
0.67 s window either side, where the positional error is far below anything visible.

Measured result, via `sequencer.get_binding_transform` at frames 1190 / 1200 / 0 / 10:

```
arriving  (P1200 − P1190)/10 frames = (+7.44, −2.375, −11.91) per frame
departing (P10   − P0)   /10 frames = (+7.44, −2.375, −11.91) per frame
```

Identical to the last digit reported, on all three axes, and the rotation channels match too. So the
technique works — but it costs two extra keys, a hand-derived velocity, and an understanding of an
undocumented engine special case that nothing in the wiki mentions. Every author of a looping
cinematic through this API will hit the same wall, and most will not diagnose it: the symptom is
"my loop stutters", and the cause is three layers down in `FRichCurve`.

## Proposed

1. **`arriveTangent` / `leaveTangent` (numbers) on the keyframe verbs**, honoured when
   `tangentMode: "user"` or `"break"`. This is the minimal, general fix, and it makes the existing
   `tangentMode` parameter mean something.
2. **Surface tangents in the readback.** `sequencer.list_sections {includeKeys:true}` now emits
   per-key `interp` (shipped by `F-sequencer-curve-channel-ops` `#2` — confirmed live at HEAD).
   Add `arriveTangent` / `leaveTangent` / `tangentMode` alongside it, otherwise a caller who sets
   tangents still cannot verify them, and the decoy-key workaround remains the only *provable*
   route.
3. Optional but the natural home for this problem: a `sequencer.close_loop {path, bindingId}` that
   copies frame 0's value to the last playback frame and sets both endpoints' tangents from the
   interior neighbours, so the common case is one call. `RCCE_Cycle` pre/post-infinity extrapolation
   on the channels would be a related, cheaper primitive worth exposing at the same time.

## Impact

Medium. Achievable today, but only by an author who already knows the engine's endpoint-tangent
rule, and only with two keys that exist purely to work around the API. A looping flythrough is one
of the most common things anyone builds with a level sequence.

## Distinct from

- `F-sequencer-curve-channel-ops` (IN-REVIEW) added `interp` and `tangentMode` **selection** and its
  readback half — verified working at HEAD. This ticket is the tangent **values** that
  `tangentMode:"user"` has no way to receive; `#2` there did not claim to cover them.
- `F-sequencer-batch-keyframes` is the call-count/atomicity gap on the same verbs, not a curve-shape
  gap.

## History

- `#1-measured` `OPEN` reporter — Hit while authoring a 20 s looping camera flythrough where the
  brief made an invisible loop seam a hard requirement. Diagnosed from `FRichCurve::AutoSetTangents`'
  endpoint special case, worked around with the two decoy keys described above, and the workaround
  verified numerically with `sequencer.get_binding_transform` at frames 1190/1200/0/10 (velocities
  matched exactly on all three position axes and on both animated rotation channels). Not filed as a
  bug: nothing misbehaves, the capability is simply absent.
