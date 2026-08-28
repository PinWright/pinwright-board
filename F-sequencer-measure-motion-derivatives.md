---
id: F-sequencer-measure-motion-derivatives
title: "No way to see whether a sequencer transform track's MOTION is any good — 47 methods report keys, values and tangents but nothing reports speed, acceleration, jerk, angular rate or path curvature, so a camera can pass every available check and still read as jagged"
status: OPEN
severity: High
category: feature
tags: [sequencer, cinematics, camera-path, motion, derivatives, curves, verification, measure_motion]
encounters: 1
---

# A transform track's authored data is fully readable; its motion is not

The `sequencer` namespace exposes ~47 methods. Between them a caller can read every *authored
number* on a 3D transform track: `list_sections {includeKeys:true}` emits per-key `frame`, `value`,
`interp`, `tangentMode`, `arriveTangent` and `leaveTangent`; `get_binding_transform` evaluates a
composited pose at one frame; `get_properties` gives tick resolution and playback range.

**Not one of them reports a derivative.** There is no verb that answers *how fast is the camera
moving*, *where does it accelerate*, *where does the path kink*, or *does the loop close in
velocity* — the questions that decide whether a cinematic reads well. Those are the properties an
author is actually trying to control, and today they are reachable only by hand-rolling: bake the
channels, difference the samples, and reimplement the Hermite basis to get anything past the first
derivative without numerical noise.

## Why this is not a nice-to-have: it hides a whole class of defect

The gap is proven by a defect that survived a rigorous verification pass on
`/Game/Atlantis/Cine/LS_Atlantis_Flythrough` (60 fps, 1201-frame loop, 24 keys).

That pass measured, and passed, all of:

- key count, key times, key values, tangent modes and tangent values, read back and checked against
  the `.uasset` bytes on disk;
- a per-frame speed profile — min 1107, max 6151, mean 2877 uu/s, no key at a near-zero minimum;
- **loop-seam velocity continuity, proved instantaneously by convergence**: the one-sided derivative
  difference at the seam fell 45.3 -> 4.55 -> 0.455 uu/s as the sampling step shrank 10x each time,
  which is first-order convergence to zero and is what C1 continuity looks like;
- per-frame clearance against level geometry, 0 frames under threshold.

The rendered result was still reported by the user as *"jagged non-smooth trajectory, that feels
bad."*

Every one of those checks measures the **first** derivative or less. The defect was in the second
and in the path's curvature, and it was invisible to all of them:

| measured afterwards, by hand | value |
|---|---|
| acceleration step at each key (C1 curves are discontinuous in acceleration by construction) | median 4970, max 13118 uu/s^2, at **23 of 23 keys** |
| segment jerk | median 10210, max 67547 uu/s^3 |
| **minimum turn radius** | **66 uu** |
| lateral (centripetal) acceleration at that corner | 10148 uu/s^2 |

A 66-uu radius of curvature on a path whose subject is 4600 uu away is a hairpin. Nothing in the
namespace reports it, and nothing in the namespace would have reported it after a fix either.

**The specific trap it hid is worth stating because the API invites it.** Shrinking a key's tangent
is the obvious way to author a deceleration — a ritardando into a hero shot — and the newly shipped
`arriveTangent`/`leaveTangent` parameters make it a one-liner. But shrinking a tangent **at a vertex
where the path also changes direction** concentrates the entire direction change into a tight corner
at that key: on this asset a 63-degree turn plus a tangent scaled 1309 -> 820 uu/s produced the
66-uu radius above. The authored numbers all read as intended; only the curvature shows the damage.
A caller with an explicit tangent parameter and no curvature readback is being handed a loaded
weapon.

## The analogue already exists in this plugin

`animation.measure_motion` does exactly this job for an `UAnimSequence`: it samples authored motion
straight from the data model — no component, no world, no playhead, no rendering — and returns
per-check `status` of `pass` / `fail` / `reported` / `unmeasured`, with a top-level `pass` that is
false whenever any check could not be measured. It already carries a `loopSeam` check and a `jitter`
check, and `looping` is a first-class parameter.

There is no reason a sequencer transform track should be less measurable than a skeletal animation.
The sampling source differs (`evaluate_keys` on the scripting channels, or the interrogation
pipeline `get_binding_transform` already uses); the output shape should not.

## Proposed: `sequencer.measure_motion {path, bindingId, ...}`

Sample the bound object's evaluated transform over the playback range and report, per check, with
the `measure_motion` status vocabulary:

1. **`speed`** — min / max / mean / ratio, and the value at every key. A key sitting at a local
   minimum approaching zero is the "stops dead at every keyframe" defect
   (`B-sequence-add-keyframe-transform-keys-never-auto-set-tangents`), which shipped once already.
2. **`acceleration`** — the step across every key, absolute and relative to the local magnitude.
   Cubic Hermite curves are C1, not C2, so a step at every key is *expected*; the useful output is
   the distribution and the outliers, so an author can ask of each one *what beat is this?* This
   check should be `reported` by default, not a gate — see below.
3. **`jerk`** — per segment (constant within a cubic segment, so it is exact, not sampled).
   **Report the segment durations alongside it**: jerk scales as `1/h^3`, so a segment half its
   neighbours' length carries 8x their jerk for the same shape. On this asset every jerk outlier was
   a spacing artefact, and the fix was re-timing, not re-shaping. A caller cannot see that without
   the duration column.
4. **`curvature`** — minimum turn radius and the frame it occurs at, plus the tangential/normal
   split of the acceleration. This is the check that would have caught the defect above, and it is
   the one with no equivalent anywhere in the plugin.
5. **`angular`** — rate and acceleration per rotation channel, in degrees. Nobody has ever measured
   these here and they can feel worse than positional jerk: on this asset yaw acceleration reached
   147 deg/s^2 at a key where the positional numbers looked ordinary.
6. **`loopSeam`** — when `looping:true`, the one-sided derivative on each side of the wrap, **at
   three sampling steps a decade apart**, and the convergence ratio. This is the only honest proof:
   a multi-frame average cannot distinguish a C1 join from a full stop — a pure zero-tangent
   smoothstep reproduces one to four decimals, which is how a stopping camera shipped here before
   (`F-sequencer-explicit-tangent-values-for-looping-cinematics` `#2`). A single-step difference is
   equally useless; the *ratio* is the evidence.

**Design note, and it is the important one: this must not be a smoothness score.** A camera with
uniformly continuous velocity and zero jerk is a floaty camera with no opinion. Real camera work
decelerates into a subject, accelerates out of a beat, holds, and cuts — those are deliberate
discontinuities. A verb that gates on low acceleration optimises the film flat and would have
rejected the best moments of this one. The useful contract is **report the profile and mark the
outliers**; the author decides which are motivated. Only `loopSeam` is a genuine pass/fail, because
a seam is a hidden cut and any mismatch there announces the join.

Suggested parameters, mirroring `animation.measure_motion`: `path`, `bindingId`, `looping` (default
true), `sampleRate` or `maxSamples`, and optional thresholds that turn individual checks from
`reported` into gated (`maxSeamVelocityMismatch`, `minTurnRadius`), off by default.

## Implementation notes from doing it by hand

- **Sample analytically, not by finite differences.** Differencing a dense bake at 6000 Hz gives
  clean first derivatives and useless second ones — value storage is float32-precision, so the noise
  is divided by `dt^2`. The Hermite basis gives all three derivatives in closed form from the keys
  and tangents already in hand, and the third is constant per segment. A hand implementation
  validated against the engine's own `evaluate_keys` bake agreed to 9.1e-5 uu over 6 channels.
- **Tangent units are value per TICK.** Same trap as
  `F-sequencer-explicit-tangent-values-for-looping-cinematics` `#3`; a derivative verb must
  multiply by ticks-per-frame before reporting anything in per-second units.
- **`evaluate_keys(range, frameRate)` is the bake primitive** and takes an arbitrary rate, so a
  10x or 100x oversample for the seam check costs nothing extra to plumb.
- The wrap for `looping` needs the angle channels' turn count added: a yaw channel that makes one
  revolution ends at `start + 360`, and the seam derivative is wrong by `360/period` without it.

## Impact

High. Achievable today **only** through `python.execute` plus a hand-written Hermite evaluator and a
hand-written cyclic-spline/curvature analysis — roughly 200 lines that every author of a scripted
camera move has to write, get right, and validate against the engine before they can trust a single
number. The verification surface that does exist actively misleads: it makes a path that is provably
correct in every reported quantity, and visibly bad in motion, look finished.

## Distinct from

- `F-sequencer-explicit-tangent-values-for-looping-cinematics` (IN-REVIEW, shipped at `#4`) — that
  is the **write** side, and it is done: `arriveTangent` / `leaveTangent` are now settable and
  readable. This ticket is the **judge** side, and shipping the write side without it is what makes
  the tangent-shrink trap above reachable in one call.
- `F-sequencer-list-channels-keyframe-readback` / `F-sequencer-curve-channel-ops` — per-key authored
  data. Those report what was written; none reports what the motion does.
- `F-sequencer-evaluate-readback` — evaluating a pose at a frame. A derivative verb could build on
  it, but sampling one pose is not the ask.
- `animation.measure_motion` — the same job for skeletal animation, and the template to mirror.

## History

- `#1-measured` `OPEN` reporter — Found while redesigning a 20 s looping camera flythrough. The
  path passed key readback, disk-byte verification, a full speed profile with no near-zero key, an
  instantaneous three-step convergence proof of loop-seam C1 continuity, and a per-frame clearance
  sweep — and the delivered render was still reported as jagged. Diagnosing it required writing an
  analytic Hermite evaluator with first/second/third derivatives plus a tangential/normal
  acceleration split, none of which any verb provides; the actual defect was a 66-uu minimum turn
  radius, caused by shrinking a tangent at a key where the path also turned. Every figure quoted
  above is from that asset. Not filed as a bug: nothing misbehaves, the measurement capability is
  simply absent.
