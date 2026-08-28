---
id: B-sequence-add-keyframe-transform-keys-never-auto-set-tangents
title: "Transform keys from sequence.add_keyframe are stamped RCIM_Cubic + RCTM_Auto but their tangents are never computed, so every key keeps ArriveTangent/LeaveTangent 0 — the curve degrades to a chain of zero-tangent smoothsteps and the camera stops dead at EVERY key, not just the endpoints"
status: IN-REVIEW
severity: Critical
category: bug
tags: [sequencer, add_keyframe, transform-track, tangents, curves, cinematics, camera-path, silent-corruption, motion]
encounters: 2
lastSeen: 2026-08-28T09:35:00+05:00
---

# `RCTM_Auto` without `AutoSetTangents()` is `RCTM_None` with extra steps

The transform branch of the frame-numbered `sequence.add_keyframe` stamps every key
`InterpMode = RCIM_Cubic`, `TangentMode = RCTM_Auto`, then writes it through the **raw channel
data** path:

```
Handlers/Sequencer/SequenceHandler.cpp:2186-2192
    FMovieSceneDoubleValue KeyValue(InValue);
    KeyValue.InterpMode  = KeyInterpMode;    // cubic
    KeyValue.TangentMode = KeyTangentMode;   // auto
    // KeyValue.Tangent is left default-constructed = all zeros

Handlers/Sequencer/SequenceHandler.cpp:2276-2329   Channels[n]->GetData().AddKey(TickFrame, MakeDoubleKey(v));
Handlers/Sequencer/SequenceHandler.cpp:2416        Channels[ChannelBase+i]->GetData().AddKey(...);
```

`TMovieSceneChannelData::AddKey` inserts the struct verbatim. It does not, and cannot, compute
tangents — `AutoSetTangents()` is on the channel, not on its data view. The only writers that reach
it are the channel's own typed adders (`MovieSceneDoubleChannel.cpp:218, 229, 262, 268`, entered via
`AddCubicKey` at `:157-160`), and this branch never goes through them.

So `RCTM_Auto` is recorded as an intent that nothing acts on, and `ArriveTangent` / `LeaveTangent`
stay **0.0 on every key forever**. Evaluation reads the stored tangents directly, so a cubic key
with zero tangents is a flat-in/flat-out Hermite and the channel becomes a chain of independent
smoothsteps.

## What it does to a shot

The camera **decelerates to a standstill at every keyframe** and accelerates out of it again. Not
"eases" — stops. A path authored as one continuous move plays as N-1 separate moves.

Measured on `/Game/Atlantis/Cine/LS_Atlantis_Flythrough` — 23 keys, 60 fps, frames 0-1200, a 20 s
flythrough meant to read as one unbroken flight. Speed sampled from
`MovieSceneScriptingDoubleChannel.evaluate_keys` over all 1201 display frames:

| key frame | 0 | 40 | 130 | 250 | 380 | 480 | 560 | 640 | 700 | 740 | 780 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| speed uu/s | 101 | 33 | 25 | 24 | 34 | 47 | 54 | 109 | 266 | 378 | 448 |

| key frame | 820 | 860 | 880 | 900 | 940 | 970 | 1010 | 1050 | 1085 | 1116 | 1160 | 1200 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| speed uu/s | 511 | 953 | 995 | 511 | 707 | 527 | 875 | 1348 | 1286 | **92** | **101** | **101** |

Against **per-segment peaks of 1,500 - 24,043 uu/s**. Every one of the 23 keys is a local minimum of
speed, most by one to two orders of magnitude. Read straight off the asset, all 23 keys of all six
animated channels report:

```
tangentMode = RCTM_AUTO,  arriveTangent = 0.0,  leaveTangent = 0.0
```

## The two call shapes disagree, which is the tell

`sequencer.add_keyframe`'s other registered shape (seconds-based, float property tracks,
`Handlers/Sequencer/SequencerHandler.cpp:195`) writes via `Channel.AddCubicKey(FrameNumber, Value,
TangentMode)`, which **does** reach `AutoSetTangents()`. The same nominal request therefore produces
a smooth curve on a float property track and a stuttering one on a transform track, purely as a
function of which handler the dispatcher picked. Neither response says so.

## Why this is Critical

- **Silent.** Every published signal is clean: `keyCount` is right, values are right, `interp` reads
  back `cubic`, `tangentMode` reads back `auto`. `list_sections {includeKeys:true}` does not emit
  tangent values, so no readback on the surface can show the fault.
- **Misdirecting.** The symptom is "the motion looks wrong", so the author edits the keys — which
  are correct. The cause is a struct field nobody wrote.
- **Transform tracks are the common case.** This is how every RPC-authored camera move is built.

## It invalidates the documented loop-seam workaround

`F-sequencer-explicit-tangent-values-for-looping-cinematics` prescribes two "decoy" keys at frames
40 and 1160 to supply the seam velocity the forced-zero endpoint tangents cannot, and verifies with

```
arriving  (P1200 - P1190)/10 = (+7.4375, -2.3750, -11.9062) per frame
departing (P10    - P0)  /10 = (+7.4375, -2.3750, -11.9062) per frame
```

Those numbers are reproduced **exactly** by a pure zero-tangent smoothstep over the 40-frame f0-f40
span: `h(0.25) = 3(0.25)^2 - 2(0.25)^3 = 0.15625`, and
`0.15625 x (476, -152, -762) / 10 = (7.4375, -2.375, -11.9062)`.

They are a 10-frame **average across an ease**, not a seam velocity. Instantaneous:

```
P1    - P0    = (+0.8776, -0.2803, -1.4049)   ~ 53 uu/s
P1200 - P1199 = (+0.8776, -0.2802, -1.4049)   ~ 53 uu/s
```

The camera still arrives at the seam at rest and leaves at rest — the exact hitch the decoy keys
were added to remove. The mirror makes the stop *symmetric*, so the loop does not visibly jump, but
it does not make it moving. The workaround cannot work while this bug stands: adding keys near an
endpoint cannot change the endpoint's tangent, and here it cannot change any interior key's tangent
either, because none are ever computed.

## Expected

A key stamped `RCTM_Auto` has auto tangents. UE's endpoint rule then applies as designed (first
key's leave tangent and last key's arrive tangent forced to 0), which is what the decoy-key
technique is written against.

## Suggested fix

1. In the transform branch of `SequenceHandler.cpp`, collect the touched channel indices and call
   `Channels[i]->AutoSetTangents()` once per channel before `SendSuccess`. Same shape in the generic
   float branch at `:2469`.
2. Prefer the typed adders (`AddCubicKey` / `AddLinearKey` / `AddConstantKey`) over
   `GetData().AddKey` — they carry the tangent contract, so the fix becomes structural rather than a
   remembered extra call. **This is the same line that needs `UpdateOrAddKey` semantics for
   `B-sequence-add-keyframe-duplicates-existing-frame`; one edit closes both.**
3. Regression test: author three cubic/auto keys at frames 0 / 50 / 100 with values 0 / 100 / 0 on a
   transform track's `Location.Z`, then assert the **middle** key's stored `ArriveTangent` /
   `LeaveTangent` are non-zero (UE gives it the prev-to-next slope) while first and last are zero.
   Pre-fix all three are zero, so it fails before the fix — differential proof.
4. Emit `arriveTangent` / `leaveTangent` / `tangentMode` from `BuildChannelKeysJson` under the
   existing `includeKeys` gate, so this class of fault is visible from a readback at all.

## Workaround available today

UE's Python sequencer scripting API reaches what the RPC surface does not.
`MovieSceneScriptingDoubleChannel` exposes `get_keys()`, `add_key`, **`remove_key`**, and each key
carries `get_value`/`set_value`, `get_time`/`set_time`, `get_tangent_mode`/`set_tangent_mode`,
`get_arrive_tangent`/`set_arrive_tangent`, `get_leave_tangent`/`set_leave_tangent`. Via
`python.execute` that is a complete read-modify-write over an existing curve — no duplication risk,
no track rebuild. Verified: `set_value` changes only the value, leaving key count, key times and
tangent mode untouched.

## Distinct from

- `F-sequencer-explicit-tangent-values-for-looping-cinematics` (Medium, feature) is the missing
  *capability* to supply tangent values. This is a *bug*: the tangent mode the caller can already
  select is not honoured. Fixing this makes that ticket's workaround start working; fixing that one
  does not fix this.
- `F-sequencer-curve-channel-ops` (IN-REVIEW) shipped interp/tangent-mode **selection** and the
  per-key `interp` readback. It delivered the parameter; this ticket is that the transform path
  never acts on it.
- `B-sequence-add-keyframe-duplicates-existing-frame` (High) is the other defect on the same three
  source lines — `AddKey` instead of `UpdateOrAddKey`. Same call site, different property.

## Environment

UE 5.8, `EAContentExamples58`, `/Game/Maps/Atlantis`, 2026-08-27. Sequence
`/Game/Atlantis/Cine/LS_Atlantis_Flythrough`, display rate 60, tick resolution 24000, playback
`[0, 1200]`. Binding `C35BAEA544B50124B4EA1B8B9313991C` (`ACineCameraActor` possessable), track
`MovieScene3DTransformTrack_0`, section `MovieScene3DTransformSection_0`, 23 keys per channel on
`Location.X/Y/Z` and `Rotation.X/Y/Z`.

## History

- `#1-measured` `OPEN` reporter — Found while retuning the last third of the 20 s Atlantis
  flythrough. Dumped all 23 keys of all six channels via
  `unreal.MovieSceneScriptingDoubleChannel.get_keys()`: every key `RCTM_AUTO` with
  `arriveTangent == leaveTangent == 0.0`. Evaluated the six channels over all 1201 display frames
  with `evaluate_keys` and differenced them — speed collapses to a local minimum at all 23 key
  frames (24-1348 uu/s) against segment peaks up to 24,043 uu/s. Traced to
  `SequenceHandler.cpp:2186-2192` (zero-initialised `FMovieSceneTangentData`) plus the
  `GetData().AddKey` writes at `:2276-2329` and `:2416`, none of which reach `AutoSetTangents()`;
  contrasted against `SequencerHandler.cpp:195`, the sibling call shape, which does. Also showed the
  decoy-key verification in `F-sequencer-explicit-tangent-values-for-looping-cinematics` is
  reproduced to four decimal places by the zero-tangent smoothstep, so that workaround does not in
  fact produce seam velocity.
- `#2-fixed` `IN-REVIEW` developer — "Changed both transform branches and the generic float branch
  of `sequence.add_keyframe` in `Handlers/Sequencer/SequenceHandler.cpp` to call
  `FMovieSceneDoubleChannel::AutoSetTangents()` (resp. `FMovieSceneFloatChannel::AutoSetTangents()`)
  on every touched channel before responding — the transform writes now collect their channels in a
  `WriteDoubleKey` helper and solve each once after the whole write, so `RCTM_Auto` is acted on
  instead of merely recorded. Changed `BuildChannelKeysJson` in `Utils/MovieSceneJsonUtils.h` to
  emit `tangentMode`, `arriveTangent` and `leaveTangent` per curve-channel key under the existing
  `includeKeys` gate (new `TangentModeToString`), so the fault is visible from a readback;
  `Docs/wiki-src/sequencer.md` updated to match. Regression test
  `PinWright.Sequencer.SequenceAddKeyframe.CubicAutoKeysGetComputedTangents` in
  `Tests/Sequencer/TestSequenceAddKeyframeInterp.cpp`. Correction to suggestion 3: the sketched
  0/100/0 values do NOT work as a differential — UE's default auto-tangent mode
  (`Sequencer.AutoTangentNew` = 2, `MovieSceneCurveChannelImpl.cpp:770-774`) flattens any key not
  strictly between its neighbours, so the apex is legitimately 0/0 and the test would pass before
  and after. The test uses a monotonic 0/1000/3000 ramp on `Location.Z` at frames 0/50/100 instead;
  the middle key's tangents are 0 pre-fix and non-zero post-fix."
- `#3-severity-raised-to-critical` `IN-REVIEW` reporter — Raised **High → Critical**. Not because the
  mechanism is worse than `#1` describes, but because nothing in a normal, thorough process catches
  it:
  - **It is the default path for the API's primary cinematic use case.** Every RPC-authored camera
    move is a transform track built with `sequence.add_keyframe`. All of them get zero tangents.
    There is no opt-out, no flag, and no alternative verb that avoids it.
  - **The output is plausible, not broken.** A crash is loud and halts the pipeline. This produces
    clean frames, correct poses, correct key counts and a correct-length video — and ships.
  - **No read verb reveals it.** The keys report exactly what was asked for: `RCTM_Auto`. Detecting
    the fault requires tangent *values*, which the surface does not expose. This ticket and
    `F-sequencer-explicit-tangent-values-for-looping-cinematics` therefore conspire — the bug is
    invisible through the same API that causes it.
  - **Zero diagnostic.** Nothing warns, logs, or degrades.
  - **It defeated a complete review pipeline and shipped.** `/Game/Atlantis/Cine/LS_Atlantis_Flythrough`
    passed two frame-by-frame acceptance passes, a full true-16:9 re-verification, per-frame tone and
    luminance statistics, and a pixel-diff loop-seam proof. The pulsing camera reached the delivered
    1080p video anyway, where the user identified it within seconds of watching. Every one of those
    checks was a per-frame **still**; the defect exists only in the derivative, so none of them could
    have seen it.

  That last point is the argument. Severity is detectability as well as impact, and a defect that
  survives that much diligence is not a High.

  **The `#1` speed figures understate it.** Those were 1-display-frame differences, which already
  average across the trough. Re-measured by sampling all six channels at 600 Hz (10x subframe) via
  `evaluate_keys` and central-differencing:

  | | `#1` (1-frame diff) | instantaneous (600 Hz) |
  |---|---|---|
  | speed at the 23 keys | 24 - 1348 uu/s | **2.5 - 86 uu/s** |
  | global minimum | ~92 uu/s | **0.0 uu/s** (loop seam) |
  | global maximum | 24,043 uu/s | 25,560 uu/s (frame 1138) |

  Every one of the 23 keys is a local minimum, and the seam is a literal dead stop. Confirms `#2`'s
  correction about `Sequencer.AutoTangentInterpolation = 2`: the cvar defaults to 2
  (`MovieSceneCurveChannelImpl.cpp:21`), and mode 2 flattens any key not strictly between its
  neighbours, so a 0/100/0 apex is legitimately 0/0 and the monotonic ramp in the shipped test is the
  right differential.

  **Field-repaired on the asset via the Python route** (this build still has the bug; the `#2` fix is
  unverified here). Removed the frame-1160 decoy key, which let `FMovieSceneDoubleChannel::DeleteKeys`
  run `AutoSetTangents()` natively over each channel, then wrote explicit `RCTM_User` tangents on the
  two seam keys. Result on the same asset: key speeds **1008 - 6393 uu/s**, global minimum
  **577 uu/s**, peak halved to 11,951 uu/s, and the seam velocity continuous to the stored tangent.
  Details and the exact calls in `F-sequencer-explicit-tangent-values-for-looping-cinematics` `#3`.
