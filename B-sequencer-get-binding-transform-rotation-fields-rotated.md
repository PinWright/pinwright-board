---
id: B-sequencer-get-binding-transform-rotation-fields-rotated
title: "sequencer.get_binding_transform labels the rotation triple one position out — it reports (pitch, yaw, roll) under the keys (roll, pitch, yaw), so a camera's orientation reads back plausible and wrong on the namespace's only evaluation verb"
status: IN-REVIEW
severity: High
category: bug
tags: [sequencer, get_binding_transform, rotation, frotator, silent-wrong-data, readback, verification-verb, camera, transform-track]
encounters: 2
lastSeen: 2026-08-27T20:50:00+05:00
---

# The rotation readback is shifted by one field

`sequencer.get_binding_transform` returns `rotation: {roll, pitch, yaw}`. The three numbers are the
right numbers in the wrong slots: what comes back as `roll` is the **pitch**, what comes back as
`pitch` is the **yaw**, and `yaw` is always `0` (it is carrying the roll, which happened to be 0
here).

Location and scale are correct. Only the rotation triple is affected.

## Evidence

Authored on `/Game/Atlantis/Cine/LS_Atlantis_Flythrough`, a 60 fps / 0-1200 level sequence with a
`MovieScene3DTransformTrack` on a possessed `ACineCameraActor`, keys written with the legacy
frame-based `sequence.add_keyframe`.

`sequencer.list_sections {includeKeys: true}` — the authoritative read of what is stored —
shows the six rotation-bearing channels in engine order `X, Y, Z, roll, pitch, yaw`:

| frame | ch3 (roll) | ch4 (pitch) | ch5 (yaw) |
|---|---|---|---|
| 0    | 0 | -22   | 0     |
| 760  | 0 | *interp* | *interp* |
| 1200 | 0 | -22   | 360   |

`sequencer.get_binding_transform` on the same binding at the same frames:

| frame | reported `roll` | reported `pitch` | reported `yaw` |
|---|---|---|---|
| 0    | **-22** | 0     | 0 |
| 1200 | **-22** | **360** | 0 |
| 1190 | -22.148 | 359.925 | 0 |

Frame 0 alone is ambiguous (roll and yaw are both 0), which is why frame 1200 matters: the stored
yaw is `360` and it surfaces under the key `pitch`.

The interpolated case pins it without relying on any endpoint. Keys either side of frame 760 are
`f740 {pitch: 2, yaw: 0}` and `f780 {pitch: 11, yaw: 38.3}`, so the evaluated pose at 760 must be
roughly `pitch 6.5, yaw 19.15, roll 0`. The response:

```json
{"frame":760, "location":{"x":-4639,"y":-1611,"z":780},
 "rotation":{"roll":6.5, "pitch":19.15, "yaw":0}}
```

6.5 is the pitch and 19.15 is the yaw, both one slot to the left. Three frames, one consistent
one-position rotation of the field names.

## Why this is worse than a cosmetic naming bug

`get_binding_transform` is the *verification* verb for this namespace — the wiki sells it as the
thing that reports "the transform Sequencer would apply at an arbitrary (possibly non-key) frame",
as distinct from `list_tracks`/`list_sections`, which only return authored keys. It is therefore
what an author reaches for to answer "where is my camera actually pointing at frame N, after
easing and section blending". The answer it gives is well-formed, self-consistent, and wrong.

For a camera it is maximally confusing, because a camera's pitch is the field an author checks most
and `roll` is the field they expect to be 0. Reading `roll: -22` on a camera that has no roll reads
as a real defect in the authored data, and sends the author to fix a curve that is already correct.
Going the other way is worse: an author who wants pitch and reads the `pitch` key gets the yaw, and
a yaw of 360 looks like a plausible pitch value in a sequence that legitimately sweeps 360.

The one case it is silently harmless is the common one — roll 0, yaw 0 — which is exactly why it
can survive a long time.

## Likely cause

The handler is almost certainly reading the transform section's three rotation channels by index
(`Rotation.X`, `Rotation.Y`, `Rotation.Z`) and emitting them into a `{roll, pitch, yaw}` object with
an off-by-one in the mapping, or building an `FRotator` from `FRotator(X, Y, Z)` when the section's
channel order is roll=X, pitch=Y, yaw=Z and `FRotator`'s constructor is `FRotator(Pitch, Yaw, Roll)`.
The second is the more likely shape and matches the observed permutation exactly: feeding
`(roll, pitch, yaw)` into a `FRotator(Pitch, Yaw, Roll)` constructor and then printing
`.Roll/.Pitch/.Yaw` produces reported-roll = actual-yaw... which is not what is seen, so check the
emit side first: reported `roll` = actual `pitch`, reported `pitch` = actual `yaw`, reported `yaw` =
actual `roll` is a left-rotation of the emitted triple.

Whichever it is, the fix belongs next to a regression test that keys a binding with **three
distinct non-zero, non-equal** rotation components (e.g. roll 5, pitch -22, yaw 130) and asserts each
field by name. A test using a camera-shaped rotation (roll 0) cannot catch this.

## Suggested fix

1. Correct the mapping in the `get_binding_transform` emit path.
2. Regression test with three distinct rotation components, at a **non-key frame** so the
   interrogation pipeline is genuinely exercised, asserting `roll`/`pitch`/`yaw` individually.
3. While there: `list_sections` returns the transform section's ten channels as an unnamed,
   unlabelled array (`X, Y, Z, roll, pitch, yaw, scaleX, scaleY, scaleZ, weight`). Nothing in the
   response or the wiki says which index is which, so the order has to be learned by writing a probe
   key and reading it back. A `name` or `role` field per channel entry would remove that step and
   would also have made this defect obvious on first read.

## Impact

High. Silent wrong data on the namespace's only evaluation-time readback, in the field an author
checks most, on the verb the wiki nominates for exactly this job. It does not corrupt anything — the
authored curve is correct and the rendered shot is correct — but it makes the readback untrustworthy,
and the failure is invisible in the roll-0/yaw-0 case that most tests would use.

## Environment

UE 5.8, `EAContentExamples58`, `/Game/Maps/Atlantis`, 2026-08-27. Sequence
`/Game/Atlantis/Cine/LS_Atlantis_Flythrough` (display rate 60, playback 0-1200, tick resolution
24000), binding `9D3651C4497E5DA2BE143086173708E7` on `CineCam_Atlantis_Flythrough`
(`ACineCameraActor`).

## History

- `#1-measured` `OPEN` reporter — Hit while authoring a 20 s looping camera flythrough and using
  `get_binding_transform` to prove velocity continuity across the loop seam. Cross-checked against
  `sequencer.list_sections {includeKeys:true}` on the same binding, which reports the stored channel
  values and disagrees. Confirmed at three frames including one non-key frame (760) whose value is
  produced by interpolation, so the permutation is in the readback and not in the authored data. The
  positional half of the same response is correct and was used, unchanged, to verify the seam.

- `#2-independent-confirmation-from-world-geometry` `OPEN` reporter — 2026-08-27, UE 5.8, same asset
  (`/Game/Atlantis/Cine/LS_Atlantis_Flythrough`), reached from a different direction and without
  reading `list_sections`. A VFX placement agent sampled `get_binding_transform` at 20 frames to
  recover the flight path before siting bubble vents, and the reported orientations were checked
  against world geometry instead of against stored keys: through the temple orbit the camera must
  look at the origin, so the true yaw at each frame is the bearing from the sampled location to
  (0, 0). It matches the field labelled **`pitch`** at every orbit frame, to within a degree:

  | frame | reported location | bearing to origin | reported `pitch` | reported `roll` |
  |---|---|---|---|---|
  | 780  | (-4078, -3222) | 38.3   | **38.3**   | 11    |
  | 810  | (-1802, -5430) | 71.6   | **70.6**   | 6.78  |
  | 840  | (699, -5913)   | 96.7   | **95.75**  | -4    |
  | 870  | (3682, -5367)  | 124.4  | **124.45** | -14   |
  | 900  | (6237, -3167)  | 153.1  | **153.1**  | -13   |
  | 930  | (6683, 656)    | 185.6  | **185.42** | -11.3 |
  | 960  | (5380, 3407)   | 212.3  | **212.66** | 17.9  |
  | 990  | (3173, 5689)   | 240.9  | **239.4**  | 17.5  |

  The residual `roll` values (-14 .. +18) then read correctly as pitch for a camera at Z 900-4400
  looking at a temple whose ridge is Z 4424. So the permutation is confirmed by an oracle entirely
  outside Sequencer: the field named `pitch` carries yaw, the field named `roll` carries pitch.

  Cost, and why this is worse than a mislabel: the numbers are individually plausible in the slots
  they arrive in. A camera path read as `pitch: 153.1` looks like a camera pointed nearly straight
  down (and 239.4 looks like a wrapped -120). The agent's first pass composed effect placements
  against that reading before the geometry cross-check caught it; every effect would have been
  sited behind the camera. `get_binding_transform` is the only verb that evaluates a binding at an
  arbitrary frame, so there is no second evaluation verb to disagree with it - the cross-check has
  to come from `list_sections`' stored keys or, as here, from outside the sequence entirely.

- `#3-read-through-property-composites` `IN-REVIEW` developer — Root cause is engine-side, not in the
  emit path: `FInterrogationChannels::QueryLocalSpaceTransforms`' fully-animated fast path builds its
  result as `FIntermediate3DTransform(LocX, LocY, LocZ, RotY, RotZ, RotX, ...)`
  (`MovieSceneInterrogationLinker.cpp`, identical in UE 5.3-5.8), so Rotation.Y lands in `R_X`,
  Rotation.Z in `R_Y` and Rotation.X in `R_Z`; since `GetRotation()` returns `FRotator(R_Y, R_Z, R_X)`
  the triple surfaces rotated one slot left, exactly as measured. The partially-animated path in the
  same function assigns `R_X/R_Y/R_Z` straight from `DoubleResult[3..5]` and is correct, so a fixed
  re-shuffle on our side would break masked-channel sections. `sequencer.get_binding_transform` now
  keeps `QueryLocalSpaceTransforms` only as an emptiness probe (eval-disabled track) and reads the
  values back with `FSystemInterrogator::QueryPropertyValues(ComponentTransform, ...)`, which maps
  `DoubleResult[3] -> R_X, [4] -> R_Y, [5] -> R_Z` as `ComponentTransform` declares them and is
  path-independent. Response shape unchanged; location/scale unchanged. Files:
  `Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp` (get_binding_transform only).
  Test added: `PinWright.Sequencer.EvaluatedReadback.RotationFieldsMatchAuthoredComponents` in
  `Source/PinWright/Private/Tests/Sequencer/TestSequencerBindingTransformRotationFields.cpp` — keys
  roll/pitch/yaw as three distinct non-zero values via linear ramps and asserts each component under
  its own key at a non-key frame; pre-fix all three assertions fail. Not compiled or run (orchestrator
  owns builds). Untouched: `list_sections`' unlabelled channel array (ticket suggestion 3) — separate
  ticket.
