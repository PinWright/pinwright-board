---
id: B-verify-grounding-maxgap-counts-overhangs
title: "`spatial.verify_grounding` gates `pass` on `maxGapCm`, which measures the highest overhanging geometry rather than the seating, so anything with a capital, a flank or an overhang fails with a reason that is false — and the one genuine failure in the batch is indistinguishable from the eight false ones"
status: OPEN
severity: High
category: bug
tags: [spatial, verify-grounding, ground-actors, max-gap, footprint-columns, false-failure, overhang, curvature, misleading-failreason]
encounters: 1
lastSeen: 2026-08-27T19:15:37+05:00
---

# The float check measures the top of the actor, not the bottom

`spatial.verify_grounding` reports a correctly seated actor as
`pass: false`, `failCode: "ACTOR_NOT_GROUNDED"`,
`failReason: "Part of this actor floats 2327.90 cm above the surface (max allowed 2.00 cm)."`

The actor is an intact Doric column standing on flat terrain with its base bedded **48 cm
into** the ground. The 2327.90 cm is the underside of its **capital**, 2352 uu up, which is
supposed to be in the air. On that same actor, from that same call:

```
minGapCm: -48.43    coverage: 1    contactPoints: 4    groundSpreadCm: 0    averageNormal (0,0,1)
```

Every field that describes the seating says it is seated. One field that describes the
*silhouette* says it is floating, and that is the field `pass` is gated on.

## Repro — measured 2026-08-27, `/Game/Maps/Atlantis`, UE 5.8

Thirteen avenue columns, all seated first by:

```
spatial.ground_actors {surface:{preset:"landscape"}, prefix:"AVE_Col_", samples:4,
                       seatPercentile:0, embedFraction:0.02}
  -> placed: 13, failed: 0
```

then verified:

```
spatial.verify_grounding {surface:{preset:"landscape"}, prefix:"AVE_Col_",
                          samples:4, maxPenetration:60, detail:"all"}
  -> pass:false, checked:13, passed:5, failed:8
```

## Controlled experiment: the mesh is the only variable

Same pad, same seating call, same terrain, same verify call. The only thing that differs
between the rows is which mesh the actor uses:

| actor | mesh | overhang? | `minGapCm` | `contactPoints` | `maxGapCm` | `pass` |
|---|---|---|---|---|---|---|
| AVE_Col_N1_intact | `SM_Column_Doric` | capital, 370 wide over a 360 base | -48.43 | 4 | **2327.90** | false |
| AVE_Col_N4_intact | `SM_Column_Doric` | same | -48.47 | 5 | **2269.23** | false |
| AVE_Col_N6_intact | `SM_Column_Doric` | same | -48.40 | 5 | **2266.82** | false |
| AVE_Col_S0_intact | `SM_Column_Doric` | same | -48.45 | 6 | **2272.76** | false |
| AVE_Col_N3_snapped | `SM_Column_Broken_A` | none — snapped off | -29.99 | 5 | -22.21 | **true** |
| AVE_Col_S1_snapped | `SM_Column_Broken_A` | none | -30.10 | 6 | -13.39 | **true** |
| AVE_Col_N0_stump | `SM_Column_Broken_B` | none | -12.81 | 4 | -2.95 | **true** |

`groundSpreadCm: 0` and `averageNormal (0,0,1)` on every row. **Snap the capital off and
the same column passes.** The ones that fail are exactly the ones whose mesh has geometry
overhanging its own footprint.

### Second mechanism, same root cause: curvature

`SM_Column_Toppled` is a 2083-long shaft lying on its side, seated to `contactPoints: 6..8`,
`penetrationCm: 22..33`, `groundSpreadCm: 0..24` — bedded into flat sand along its whole
length. It reports `maxGapCm: 76.27 / 82.20 / 95.65`.

That is not a floating segment, it is the **flank of the cylinder**. With the default
`footprintInset: 0.1` the outermost sample column sits at 90% of the half-width, and for a
circle of radius 147 the underside there is

    147 - 147 * sqrt(1 - 0.9^2)  =  147 * (1 - 0.4359)  =  83 uu

off the ground. The measured 76-96 band brackets that 83. Any lying cylinder, barrel, log,
pipe or dome reports the same and can never satisfy `maxGapCm <= 2`.

### Cross-checks that rule out a real seating fault

- `spatial.raycast {origin:{x:-9000,y:1600,z:4000}, direction:(0,0,-1), onlyClasses:["LandscapeProxy"]}`
  -> `z: 0.0`, `normal (0, -0.023, 0.9997)`. The pad really is flat, so there is nothing
  under those columns for a 2327 cm gap to be measured against.
- `spatial.ground_actors`' own post-move check, which gates on `contactPoints`/`seatError`
  rather than `maxGap`, reported `placed: true, seatErrorCm: 1.1e-13` for every one of them.

Two verbs measuring the same actors on the same frame disagree, and the one that disagrees
is the one whose criterion reads the silhouette.

## Correction to the original report — `maxGap` IS a parameter

The defect log this came from claimed "`maxGap` is not a parameter of `verify_grounding` at
all, so a caller cannot loosen the check." **That is false in this tree** and is recorded
here so it does not propagate. Verified at HEAD:

```cpp
// Handlers/Spatial/GroundPlacementHandler.cpp:875-878
GroundRpcParam(TEXT("maxGap"), TEXT("number"),
    TEXT("Largest allowed clearance in cm between any footprint column and the surface. "
         "This is the floating check."),
    TEXT("2"), TArray<FString>({TEXT("max_gap")})),
```

`maxGap` and `maxPenetration` are both declared, both default `"2"`, both carry snake_case
aliases, and both are read at `GroundPlacementHandler.cpp:935-940`. The 2.0 cm allowance is
that **default**, not a constant frozen at the comparison — it falls back to
`GroundPlacement::DefaultContactToleranceCm`, `constexpr double DefaultContactToleranceCm = 2.0;`
at `Handlers/Spatial/GroundPlacementUtils.h:79`.

**The knob's existence does not fix this, and that is the stronger point.** `maxGapCm` for
an intact column is 2327.90 because the capital is 2352 uu up. A `maxGap` large enough to
admit it would have to exceed 2327 — which does not *loosen* the float check, it **switches
it off** for every actor in the batch, including the genuinely floating ones the verb exists
to catch. There is no value of `maxGap` that accepts a seated column and still rejects a
hovering one, because the number being compared is not a property of the seating.

## Where `pass` is decided

`Handlers/Spatial/GroundPlacementUtils.cpp:465-472`:

```cpp
if (Thresholds.bEnforceGapBounds)
{
    if (Report.MaxGapCm > Thresholds.MaxGapCm)
    {
        Report.FailReasonCode = ErrorCodes::ERR_ACTOR_NOT_GROUNDED;
        Report.FailReason = FString::Printf(
            TEXT("Part of this actor floats %.2f cm above the surface (max allowed %.2f cm)."),
            Report.MaxGapCm, Thresholds.MaxGapCm);
        return;
    }
```

`bEnforceGapBounds` is forced `true` for this verb — `GroundPlacementHandler.cpp:950`,
`Thresholds.bEnforceGapBounds = true;` under the comment "This verb asks the absolute
question, so the gap bounds ARE the criteria here." `bPass = true` is reached only by
falling off the end of the check chain at `GroundPlacementUtils.cpp:508`. `ACTOR_NOT_GROUNDED`
has a second, legitimate emit site at `:448` for zero contact points. Serialization:
`pass` at `GroundPlacementHandler.cpp:446`, `failReason` at `:450`, batch
`pass = FailCount == 0` at `:988`.

## Impact: the false failures bury the real one

On this batch **8 of 13 failed and all 8 were false.** The batch did contain one genuine
defect — three shafts balanced on one end, `contactPoints: 1`, `groundSpreadCm: 300`, real
`maxGapCm` 451 — and it was flagged through the **same field** with the **same
`failCode`**, so `pass`/`failCode` alone cannot separate it from the eight. Telling them
apart required reading `contactPoints` and `groundSpreadCm` by hand, which is exactly the
box-math the verb exists to replace.

The verb is the published way to prove a batch is seated (`spatial.ground-placement`
§ Verifying, and its own § Suggested workflow step 3: "Anything still failing has a real
reason attached to it"). For a level of columns, statues, toppled shafts and boulders — the
normal contents of a ruin — the majority of actors fail with a reason that is not real, and
the sentence quoted in that workflow stops being true. The false reason is also
actionable-looking: "part of this actor floats 2327.90 cm" is a statement about a capital
doing exactly what a capital does, and it reads as a placement bug. It cost four re-seat
cycles here before the snapped-vs-intact control isolated it. It does not take the editor
down.

## Fix, in order of preference

**(a) Measure the float against the actor's supported columns only** — the columns whose
underside geometry is within some fraction of the actor's lowest point — so a capital
2352 uu up and a cylinder flank 83 uu up stop voting.

**This is already half-built.** The verb computes per-column clearances and already
distinguishes the two column populations: `actorColumns` and `supportedColumns` are separate
fields, serialized side by side at `GroundPlacementHandler.cpp:461-462`, and used separately
in the coverage message at `GroundPlacementUtils.cpp:433`
(`Report.Coverage * 100.0, Report.SupportedColumns, Report.ActorColumns`). Restricting the
`MaxGapCm` reduction to the supported set needs no new sampling and no new data — only a
different set to take the max over.

**(b)** Report both — keep the existing silhouette figure under a name that says so
(`maxColumnClearanceCm`) and gate `pass` on a new seating-scoped `maxGapCm`. This preserves
the diagnostic while fixing the verdict.

**(c) At minimum, reword the `failReason`** so it names what was actually measured, and say
in the message that overhanging geometry counts. A caller who reads "the highest point of
this actor's underside profile is 2327.90 cm above the surface" will not spend four cycles
re-seating.

Note that (b) and (c) alone leave the batch signal buried; only (a) restores the
"anything still failing has a real reason attached to it" contract.

## Workaround

Ignore `pass` and `maxGapCm` for anything that is not flat-bottomed. Gate on `coverage`,
`contactPoints >= 3`, and `penetrationCm` within the intended embed. For the genuinely
balanced case gate on `groundSpreadCm` being small relative to the actor's footprint, or
re-seat with a higher `seatPercentile` and confirm `contactPoints` rises. That is what
separated the three real failures from the eight false ones here — and it means discarding
the verb's headline output and re-deriving the verdict from its detail fields.

## Cross-verb datum

The sibling verb already carries this knob as a **caller expectation**:
`B-trace-complex-hits-render-geometry` line 75 records a passing
`spatial.verify_placement {expect:{grounded:{maxGap:2}}}` -> `pass: true`, gap ~1.7e-13, as
a no-regression control. So `maxGap: 2` is an established, working assertion on a
single-actor flat-bottomed probe cube. That is the shape the threshold was designed for, and
it is exactly the shape a ruin does not have.

## Dedup

The strings `verify_grounding`, `ground_actors`, `maxGapCm`, `maxPenetration`,
`seatPercentile`, `footprintInset`, `contactPoints`, `groundSpreadCm`, `seatError` and
`embedFraction` appear nowhere else on this board. The only two `spatial.*` tickets
(`F-spatial-raycast-no-batch-multi-origin`, plus the `spatial.verify_placement` control line
in `B-trace-complex-hits-render-geometry`) do not touch grounding verdicts.

severity rationale: impact=wrong verdict with a false, actionable-looking reason on the verb's normal and documented path — the caller trusts "part of this actor floats 2327.90 cm above the surface" and re-seats an actor that is correctly seated, and because the genuine failure carries the identical field and `failCode`, `pass` stops separating real from false (High) x reach=this is the published verification step for every batch grounding, and it misfires on every actor that is not flat-bottomed — columns, statues, toppled shafts, boulders, i.e. the normal contents of a ruin, 8 of 13 here -> High. A workaround exists but consists of discarding the verb's headline output and re-deriving the verdict from `coverage`/`contactPoints`/`groundSpreadCm` by hand, which is the manual box-math the verb was written to replace.

## History
- `#1-maxgap-measures-the-capital` `OPEN` reporter — Found while building the Atlantis example level on host project EAContentExamples58 (map as forcing function; see that project's `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. 13 avenue columns seated by `spatial.ground_actors` (`placed:13, failed:0`), then `spatial.verify_grounding {samples:4, maxPenetration:60, detail:"all"}` -> `pass:false, checked:13, passed:5, failed:8`; **all 8 failures false**. Controlled mesh-only experiment on one flat pad (`groundSpreadCm:0`, `averageNormal (0,0,1)` throughout): 4 intact `SM_Column_Doric` fail at `maxGapCm` 2266-2328 while 3 broken columns pass at -22.21/-13.39/-2.95 from the identical call — snap the capital off and the same column passes. Failing actors report `minGapCm -48.4`, `coverage 1`, `contactPoints 4-6`, i.e. every seating field says seated; the 2327.90 cm is the capital's underside 2352 uu up. Second mechanism on `SM_Column_Toppled`: `maxGapCm` 76.27/82.20/95.65 is the cylinder flank — with `footprintInset 0.1` the outermost column sits at 0.9R and `147 - 147*sqrt(1-0.81) = 83` uu predicts the measured 76-96 band, so any lying cylinder/barrel/log/dome can never satisfy `maxGapCm <= 2`. Real-fault cross-checks: `spatial.raycast` on the pad -> `z:0.0`, `normal (0,-0.023,0.9997)`; `ground_actors`' own post-move check -> `seatErrorCm 1.1e-13` for every one. **Correction to the originating log, which must not propagate:** it claimed `maxGap` is not a parameter of the verb. It IS — declared at `Handlers/Spatial/GroundPlacementHandler.cpp:875-878` (default `"2"`, alias `max_gap`) beside `maxPenetration`, read at `:935-940`, falling back to `constexpr double DefaultContactToleranceCm = 2.0` at `Handlers/Spatial/GroundPlacementUtils.h:79`. The knob's existence does not help: `maxGap` would have to exceed 2327 to admit a seated column, which switches the float check off for the whole batch rather than loosening it — no value accepts a seated column while still rejecting a hovering one, because the compared number is not a property of the seating. Gating chain verified: `GroundPlacementUtils.cpp:465-472` (the `MaxGapCm > MaxGapCm` fail with the "Part of this actor floats %.2f cm" message), `bEnforceGapBounds` forced true at `GroundPlacementHandler.cpp:950`, `bPass = true` only at `GroundPlacementUtils.cpp:508`, second `ACTOR_NOT_GROUNDED` emit at `:448` (zero contact points), serialized at `GroundPlacementHandler.cpp:446`/`:450` and batch `:988`. Impact: the batch's one genuine failure (three shafts balanced on one end, `contactPoints:1`, `groundSpreadCm:300`, real `maxGapCm` 451) carried the same field and same `failCode` as the 8 false ones and was indistinguishable by `pass`/`failCode` alone. Fix (a) is half-built — the verb already separates `actorColumns` from `supportedColumns` (`GroundPlacementHandler.cpp:461-462`, used separately at `GroundPlacementUtils.cpp:433`), so scoping the float max to supported columns needs no new data. Cross-verb datum: `B-trace-complex-hits-render-geometry` line 75 records `spatial.verify_placement {expect:{grounded:{maxGap:2}}}` -> `pass:true`, gap ~1.7e-13 on a flat-bottomed probe cube, i.e. the shape the 2 cm threshold was designed for. Worked around by gating on `coverage`/`contactPoints`/`groundSpreadCm` instead of `pass`; defect untouched.
