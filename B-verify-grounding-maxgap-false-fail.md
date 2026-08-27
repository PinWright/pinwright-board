---
id: B-verify-grounding-maxgap-false-fail
title: "`spatial.verify_grounding` derives `pass` from `maxGapCm`, the highest point of the actor's underside anywhere over its footprint — so a column with a capital or a cylinder on its side can never pass however well it is seated, and the real floating actor is indistinguishable from the false ones"
status: OPEN
severity: High
category: bug
tags: [spatial, verify_grounding, ground_actors, placement, false-negative, review-hazard, level-building, scatter]
encounters: 1
lastSeen: 2026-08-27T19:20:00+05:00
---

# An intact Doric column bedded 48 cm into flat terrain is reported `ACTOR_NOT_GROUNDED`, "part of this actor floats 2327.90 cm above the surface"

The 2327.90 cm is the underside of the column's **capital**, 2352 uu up, which is supposed to be in
the air. The same response says, about the same actor, `minGapCm: -48.43`, `coverage: 1`,
`contactPoints: 4` — every field that describes the seating says it is seated.

## Repro

Thirteen avenue columns on `/Game/Maps/Atlantis`, all seated by one call that reported
`placed: 13, failed: 0`:

```js
spatial.ground_actors {surface:{preset:"landscape"}, prefix:"AVE_Col_",
                       samples:4, seatPercentile:0, embedFraction:0.02}
spatial.verify_grounding {surface:{preset:"landscape"}, prefix:"AVE_Col_",
                          samples:4, maxPenetration:60, detail:"all"}
// -> pass:false, checked:13, passed:5, failed:8
```

**The mesh is the only variable.** Same terrain (`groundSpreadCm: 0`, `averageNormal (0,0,1)` on
every row), same seat call:

| actor | mesh | overhang? | `minGapCm` | `contactPoints` | `maxGapCm` | `pass` |
|---|---|---|---|---|---|---|
| AVE_Col_N1_intact | `SM_Column_Doric` | capital, 370 wide over a 360 base | -48.43 | 4 | **2327.90** | false |
| AVE_Col_N4_intact | `SM_Column_Doric` | same | -48.47 | 5 | **2269.23** | false |
| AVE_Col_S0_intact | `SM_Column_Doric` | same | -48.45 | 6 | **2272.76** | false |
| AVE_Col_N3_snapped | `SM_Column_Broken_A` | none — snapped off | -29.99 | 5 | -22.21 | **true** |
| AVE_Col_S1_snapped | `SM_Column_Broken_A` | none | -30.10 | 6 | -13.39 | **true** |
| AVE_Col_N0_stump | `SM_Column_Broken_B` | none | -12.81 | 4 | -2.95 | **true** |

Snap the capital off the same column and it passes.

Independent confirmation the terrain really is flat, so there is nothing for a 2327 cm gap to be
measured against:

```js
spatial.raycast {origin:{x:-9000,y:1600,z:4000}, direction:{x:0,y:0,z:-1},
                 distance:8000, onlyClasses:["LandscapeProxy"]}
// -> hit:true, location.z: 0.0, normal: (0, -0.0234, 0.9997)
```

## Second mechanism, same root cause: curvature

`SM_Column_Toppled` is a 2083-long shaft lying on its side, seated to `contactPoints: 6..8`,
`penetrationCm: 22..33`, `groundSpreadCm: 0..24` — bedded along its whole length. It reports
`maxGapCm: 76.27 / 82.20 / 95.65`. That is not a floating segment, it is the **flank of the
cylinder**: with `footprintInset: 0.1` the outermost sample column sits at 90% of the half-width, and
a circle of radius 147 is `147 - 147*sqrt(1 - 0.9^2) = 83` uu off the ground there. The measured
76-96 band brackets 83. Any lying cylinder, barrel, log, pipe or dome reports the same.

Statues behave identically: `SM_Statue_Fallen_A` on ground measured `groundSpreadCm: 0` still reports
`maxGapCm: 175.2` — the rounded body of the figure.

## Why this matters more than a cosmetic false negative

[`spatial.ground-placement`](../EAContentExamples58/Saved/PinWright/wiki/spatial.ground-placement.md)
§ *Verifying* makes this verb the published proof that a batch is seated, and its § *Suggested
workflow* step 3 says "Anything still failing has a real reason attached to it."

On this pass **8 of 13 failed and all 8 were false.** The batch also contained one *genuine* defect —
three toppled shafts resting on one end, `contactPoints: 1`, `groundSpreadCm: 300`, real
`maxGapCm: 451` — and it was flagged by the same field with the same `failCode`, so `pass` /
`failCode` could not separate the real fault from the noise. It had to be told apart by reading
`contactPoints` and `groundSpreadCm` by hand, which is precisely the box-math the verb exists to
replace.

`maxPenetration` is a declared parameter, so the *buried* half of the check is tunable. There is no
counterpart for the float half — `maxGap` is not a parameter of `verify_grounding` at all — so a
caller who understands the failure cannot loosen it.

## Suggested fix, in order of preference

1. **Measure the float against supported columns only.** The verb already distinguishes
   `actorColumns` from `supportedColumns` and already has every per-column clearance in hand; a
   column whose underside geometry sits far above the actor's lowest point (a capital 2352 uu up, a
   cylinder flank 83 uu up) should not vote on "is this actor floating".
2. Failing that, **expose `maxGap` as a parameter** beside the existing `maxPenetration`, so the
   caller can set it from the actor's own geometry.
3. At minimum, **reword the `failReason`**: "Part of this actor floats 2327.90 cm above the surface"
   is a statement about a capital doing exactly what a capital does, and it reads as a placement bug.
   It cost four re-seat cycles here before the control (snapped column vs intact column, identical
   terrain, identical call) isolated it.

## Workaround

Ignore `pass` and `maxGapCm` for anything that is not flat-bottomed. Gate on `coverage`,
`contactPoints >= 3`, and `penetrationCm` within the intended embed. For the genuinely-balanced case
gate on `groundSpreadCm` being small relative to the footprint, or re-seat with a higher
`seatPercentile` and confirm `contactPoints` rises — that is what separated the three real failures
from the eight false ones.
