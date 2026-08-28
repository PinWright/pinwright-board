---
id: F-sequencer-explicit-tangent-values-for-looping-cinematics
title: "No way to set tangent VALUES (only tangentMode), so a seamless looping camera move needs decoy keys near both endpoints — RCTM_Auto forces LeaveTangent=0 on the first key and ArriveTangent=0 on the last, which makes every RPC-authored loop stop dead at the seam"
status: OPEN
severity: High
category: feature
tags: [sequencer, keyframes, tangents, curves, looping, cinematics, camera-path, workaround-required]
encounters: 3
lastSeen: 2026-08-28T09:40:00+05:00
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

High (raised from Medium at `#3`). NOT achievable today: the decoy-key technique this section
used to rest on is retracted (`#2`, `#3`), and `AutoSetTangents` zeroes the first and last key of
every cubic/auto channel regardless of how the bug ticket is fixed — so a non-zero loop-seam
velocity cannot be authored through the RPC surface at all, only through `python.execute`. A looping
flythrough is one of the most common things anyone builds with a level sequence.

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
- `#2-workaround-does-not-work` `OPEN` reporter — The decoy-key technique in `#1` does **not**
  produce seam velocity, and its own verification numbers are the proof. `(P10−P0)/10 =
  (P1200−P1190)/10 = (+7.4375, −2.3750, −11.9062)` is reproduced to four decimals by a pure
  zero-tangent smoothstep across the 40-frame f0–f40 span: `h(0.25) = 3(0.25)² − 2(0.25)³ =
  0.15625`, and `0.15625 × (476, −152, −762) / 10 = (7.4375, −2.375, −11.9062)`. That is a
  10-frame **average across an ease**, not an instantaneous velocity — and it is symmetric for the
  trivial reason that f40 and f1160 are mirrored, whatever the tangents are. Measured
  instantaneously on the same asset: `P1−P0 = (+0.8776, −0.2803, −1.4049)` and `P1200−P1199 =
  (+0.8776, −0.2802, −1.4049)`, ≈ 53 uu/s — the camera still arrives at the loop at rest and
  leaves at rest. The mirror makes the stop *symmetric* (so the still cut is invisible) but not
  *moving*. Root cause is not the missing tangent-value parameter this ticket asks for: on this
  build **no** key on the track ever gets auto tangents at all, because the transform write path
  never calls `AutoSetTangents()` — see
  `B-sequence-add-keyframe-transform-keys-never-auto-set-tangents`. Adding keys near an endpoint
  cannot change the endpoint's tangent, and here it cannot change any interior key's tangent
  either. **This feature stays worth doing** — explicit tangent values are still the only way to
  author a non-zero seam velocity once the bug is fixed — but it should be sequenced after it, and
  the `#1` workaround should not be recommended to anyone in the meantime.
- `#3-python-route-works-feature-still-required` `OPEN` reporter — Two findings: the gap is in the
  **RPC surface only**, and this feature is **not optional** for looping content even after
  `B-sequence-add-keyframe-transform-keys-never-auto-set-tangents` lands.

  **UE's Python scripting channel sets tangent values, and it works.** So this ticket has a
  known-good implementation to mirror rather than design. Exact calls, on
  `UMovieSceneScriptingDoubleKey` objects from `section.get_all_channels()[i].get_keys()`:

  ```python
  k.set_interpolation_mode(unreal.RichCurveInterpMode.RCIM_CUBIC)
  k.set_tangent_weight_mode(unreal.RichCurveTangentWeightMode.RCTWM_WEIGHTED_NONE)
  k.set_arrive_tangent(t)          # float, see units below
  k.set_leave_tangent(t)
  k.set_tangent_mode(unreal.RichCurveTangentMode.RCTM_USER)
  ```

  Each setter is a read-modify-write of the whole `FMovieSceneDoubleValue` through
  `SetValueInChannel` (`MovieSceneScriptingChannel.h:515`), so they compose in any order, and none of
  them recomputes anything. Verified end to end on
  `/Game/Atlantis/Cine/LS_Atlantis_Flythrough`: key count, key times and key values all unchanged,
  values present in the saved `.uasset` bytes.

  **Three implementation notes for whoever adds the RPC parameter — each one is a trap:**

  1. **Units are value per TICK, not per second and not per display frame.** `AutoSetTangents`
     divides by `Times[].Value` deltas, which are tick-resolution frame numbers
     (`MovieSceneCurveChannelImpl.cpp:695`), and evaluation builds the Bezier as
     `P1 = P0 + Tangent * DX / 3` with `DX` in ticks (`MovieSceneInterpolation.cpp:753-763`). On this
     asset (tick resolution 24000, display rate 60) a tangent of 1 unit/second is stored as
     `1/24000`, and one display frame is 400 ticks. Confirmed against the engine's own output: after
     `AutoSetTangents`, `Location.X` at frame 40 reads `0.025`, which is exactly its
     10.0 uu/display-frame slope divided by 400. A parameter that takes uu/s and forgets the divide
     is wrong by 24000x and will look like a hard crash of the curve.
  2. **`RCTM_User` is what makes an explicit tangent stick.** `AutoSetTangents` only touches keys
     whose mode is `RCTM_Auto` / `RCTM_SmartAuto` (`MovieSceneCurveChannelImpl.cpp:698, 731, 752`),
     so a value written under `RCTM_Auto` is silently discarded on the next recompute — including
     `PostEditChange` on load (`MovieSceneDoubleChannel.cpp:260-263`). The verb must set the mode
     alongside the values.
  3. **`remove_key` is an undocumented lever for recomputation.** `RemoveKeyFromChannel` calls
     `Channel->DeleteKeys` (`MovieSceneScriptingChannel.h:138`), which calls `AutoSetTangents()`
     (`MovieSceneDoubleChannel.cpp:226-230`). That is currently the only way to make the engine solve
     a whole channel's auto tangents from script. A `sequencer.auto_set_tangents {path, bindingId}`
     verb would be a small, obviously-useful addition and is the missing half of this request.

  **The endpoint rule survives the bug fix, so this feature is still required.** `#2` sequenced this
  after the bug on the assumption that auto tangents would then behave "as designed". They do — and
  the design is the problem: `AutoSetTangents` unconditionally forces `LeaveTangent = ArriveTangent =
  0` on the **first** and **last** key of every cubic/auto channel
  (`MovieSceneCurveChannelImpl.cpp:689-720`). A loop seam is exactly those two keys. So once the bug
  is fixed the other 21 keys move correctly and the seam **still** stops dead. Explicit tangent
  values are the only fix for the seam, on any build.

  **Retract the `#1` decoy-key technique outright — it is now actively harmful.** Under real
  tangents the frame-1160 decoy is worse than useless: its value `(-17976, 152, 10362)` lies outside
  the interval bracketed by its neighbours, so mode-2 auto-tangent flattens it to zero
  (`MovieSceneCurveChannelImpl.cpp:770-774`), reinstating a full stop 40 frames before the seam — and
  reaching it needs 10,377 uu in 44 frames, a 14,150 uu/s lurch into a standstill. The decoy does not
  merely fail to help; it converts a smooth pull-back into a dash-and-stop.

  What the decoy actually was, in hindsight, is a **hand-built Bezier control point**. Removing it
  and setting the seam tangent to 1.5x the `f0 -> f40` chord slope puts the engine's own control
  point at `P2 = P(1200) - T*DX/3 = (-17999.8, 159.6, 10400.1)` — within 24 uu of where the decoy key
  was placed. The technique had the geometry right and the mechanism wrong.

  Measured outcome of doing it properly on the same asset (600 Hz sampling, central differences):

  | | decoy keys, zero tangents | decoy removed, explicit seam tangents |
  |---|---|---|
  | speed at keys | 2.5 - 86 uu/s | 1008 - 6393 uu/s |
  | global minimum | 0.0 uu/s (at the seam) | 577 uu/s (frame 1196, not a key) |
  | global maximum | 25,560 uu/s | 11,951 uu/s |
  | leaving frame 0 | 10.2 uu/s | 2050.2 uu/s |
  | arriving frame 1200 | 0.0 uu/s | 2050.2 uu/s |

  Seam continuity proved instantaneously rather than by the 10-frame average `#2` disproved: one-sided
  differences either side of the seam converge on the identical vector as the interval shrinks —
  `1/10` frame gives 25.4 uu/s apart, `1/50` gives 4.88, `1/200` gives 0.618, against the analytic
  `tangent * 24000 = (1071.0, -342.0, -1714.5)` stored identically on both keys. Quadratic
  convergence to zero difference is what C1 continuity looks like; a 10-frame average cannot
  distinguish it from a smoothstep, which is the whole lesson of `#2`.

  Severity raised **Medium -> High** and the `## Impact` paragraph corrected: it claimed the
  capability was "achievable today" on the strength of the `#1` decoy technique, which `#2` and this
  entry retract. With that gone, and with the endpoint zeroing surviving the bug fix, there is no
  route to a moving loop seam through the RPC surface at all — only through `python.execute`.
