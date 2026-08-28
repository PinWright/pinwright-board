---
id: B-verify-grounding-maxgap-false-fail
title: "`spatial.verify_grounding` derives `pass` from `maxGapCm`, the highest point of the actor's underside anywhere over its footprint — so a column with a capital or a cylinder on its side can never pass however well it is seated, and the real floating actor is indistinguishable from the false ones"
status: DONE
severity: High
category: bug
tags: [spatial, verify_grounding, ground_actors, placement, false-negative, review-hazard, level-building, scatter, max-gap, footprint-columns, overhang, curvature, misleading-failreason]
encounters: 3
lastSeen: 2026-08-28T10:20:00+05:00
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

## History
- `#1-maxgap-is-declared-correction` `OPEN` reporter — Appended by a second reporter working the same session's defect log, ~2 min after this ticket was committed; the duplicate I had drafted (`B-verify-grounding-maxgap-counts-overhangs`) is retired to WONTFIX pointing here, and its unique content is folded in below. (This ticket was committed without a `## History` section, which the board schema requires; this entry opens one rather than reconstructing the original filing in its author's voice — the body above is that filing.) **Correction, source-verified at HEAD in this tree:** the body's claim that "`maxGap` is not a parameter of `verify_grounding` at all" is **false**, and it came from the shared defect log, so it is likely to recur. `maxGap` IS declared — `Handlers/Spatial/GroundPlacementHandler.cpp:875-878`: `GroundRpcParam(TEXT("maxGap"), TEXT("number"), TEXT("Largest allowed clearance in cm between any footprint column and the surface. This is the floating check."), TEXT("2"), TArray<FString>({TEXT("max_gap")}))`. It sits directly beside `maxPenetration`, both default `"2"`, both carry snake_case aliases, and both are read at `:935-940`, falling back to `constexpr double DefaultContactToleranceCm = 2.0` at `Handlers/Spatial/GroundPlacementUtils.h:79` — so the 2.0 cm allowance is a **default**, not a constant frozen at the comparison. **This voids suggested fix #2 ("expose `maxGap` as a parameter") — it already exists — and it strengthens #1 rather than weakening the ticket.** The knob cannot fix this: `maxGapCm` for an intact column is 2327.90 because the capital is 2352 uu up, so a `maxGap` large enough to admit it must exceed 2327, which does not loosen the float check but switches it off for every actor in the batch including genuinely floating ones. **No value of `maxGap` accepts a seated column while still rejecting a hovering one, because the compared number is not a property of the seating** — which is exactly why the batch's one real failure (`contactPoints: 1`, `groundSpreadCm: 300`, real `maxGapCm` 451) is indistinguishable by `pass`/`failCode`. Replace rung #2 with "report both": keep the silhouette figure under a name that says so (`maxColumnClearanceCm`) and gate `pass` on a seating-scoped `maxGapCm`. **Gating chain verified** so a fixer does not have to re-derive it: the fail is `GroundPlacementUtils.cpp:465-472` (`if (Thresholds.bEnforceGapBounds) { if (Report.MaxGapCm > Thresholds.MaxGapCm) { … ERR_ACTOR_NOT_GROUNDED … "Part of this actor floats %.2f cm above the surface (max allowed %.2f cm)." } }`); `bEnforceGapBounds` is forced `true` for this verb at `GroundPlacementHandler.cpp:950` under the comment "This verb asks the absolute question, so the gap bounds ARE the criteria here"; `bPass = true` is reached only by falling off the end of the chain at `GroundPlacementUtils.cpp:508`; `ACTOR_NOT_GROUNDED` has a second, legitimate emit site at `:448` for zero contact points; serialization is `GroundPlacementHandler.cpp:446` (`pass`), `:450` (`failReason`), `:988` (batch `pass = FailCount == 0`). **Suggested fix #1 is already half-built:** `actorColumns` and `supportedColumns` are separate fields serialized side by side at `GroundPlacementHandler.cpp:461-462` and already used separately in the coverage message at `GroundPlacementUtils.cpp:433`, so scoping the float max to the supported set needs no new sampling and no new data — only a different set to take the max over. **Extra data from the duplicate:** a fourth intact column, `AVE_Col_N6_intact` (`SM_Column_Doric`), `minGapCm -48.40`, `contactPoints 5`, `maxGapCm 2266.82`, `pass false` — same pad, same call. `spatial.ground_actors`' own post-move check, which gates on `contactPoints`/`seatError` rather than `maxGap`, reported `placed: true, seatErrorCm: 1.1e-13` for every one of the thirteen, so two verbs measuring the same actors on the same frame disagree and the one that disagrees is the one reading the silhouette. **Cross-verb datum:** the sibling verb already carries this threshold as a working caller expectation — `B-trace-complex-hits-render-geometry` line 75 records `spatial.verify_placement {expect:{grounded:{maxGap:2}}}` -> `pass:true`, gap ~1.7e-13 on a flat-bottomed probe cube. That is the shape the 2 cm threshold was designed for, and exactly the shape a ruin does not have. Severity `High` concurred with and left unchanged: impact = wrong verdict with a false, actionable-looking reason on the verb's documented path, and the real failure is masked; reach = the published verification step for every batch grounding, misfiring on every actor that is not flat-bottomed.
- `#2-duplicate-consolidated` `OPEN` reporter — **Consolidation: the duplicate ticket `B-verify-grounding-maxgap-counts-overhangs` has been merged into this one and deleted from the board.** This ticket survives as the single record; `encounters` 1 -> 2 to account for the merged report. The duplicate was filed ~2 min after this one by a second reporter working the same session's defect log, after a dedup sweep that ran before this ticket existed and therefore came back clean. **Attribution caveat, stated rather than guessed:** the board records no host id for either filing — neither file ever carried `claimedBy`, and all four hosts commit through the one shared board working tree, so the git author line is identical on both and cannot discriminate between hosts. The merged report is therefore credited to “the second reporter”, not to a named host. `lastSeen` is deliberately **left at 19:20:00+05:00**: that is already the later of the two observation timestamps (the duplicate carried 19:15:37+05:00), and refreshing it to the consolidation time would assert a fresh observation that did not happen. **Most of the duplicate's unique content was already folded into `#1` above** — the source-verified `maxGap`-is-declared correction, the no-value-of-`maxGap`-works argument, the full `pass` gating chain, the `actorColumns`/`supportedColumns` references, the fourth intact-column row `AVE_Col_N6_intact` (`minGapCm -48.40`, `contactPoints 5`, `maxGapCm 2266.82`), the `ground_actors` `seatErrorCm 1.1e-13` disagreement, and the `spatial.verify_placement` cross-verb datum. **What `#1` did not carry, recorded here so deleting the file loses nothing:** **(i) The duplicate's dedup-sweep record.** `verify_grounding`, `ground_actors`, `maxGapCm`, `maxPenetration`, `seatPercentile`, `footprintInset`, `contactPoints`, `groundSpreadCm`, `seatError` and `embedFraction` appear nowhere else on the board, and the only other `spatial.*` tickets — `F-spatial-raycast-no-batch-multi-origin` and the `spatial.verify_placement` control line in `B-trace-complex-hits-render-geometry` — do not touch grounding verdicts. That sweep is still valid as a statement about the rest of the board; it missed only this ticket, which had not been committed when it ran. **(ii) Concrete replacement wording for suggested fix #3 (the reword rung).** The `failReason` should name what was actually measured and say that overhanging geometry counts — e.g. “the highest point of this actor's underside profile is 2327.90 cm above the surface”. A caller who reads that will not spend four cycles re-seating a correctly seated actor. **(iii) Rung ordering, which the fix list does not state.** Rewording and report-both leave the batch signal buried: only the supported-columns fix restores the published “anything still failing has a real reason attached to it” contract, because only it stops the false failures from carrying the same `failCode` as the real one. Report-both and reword are worth shipping, but neither is a substitute. **(iv) Full observed response fields.** The verdict as returned is `pass: false`, `failCode: "ACTOR_NOT_GROUNDED"`, `failReason: "Part of this actor floats 2327.90 cm above the surface (max allowed 2.00 cm)."` — the body above quotes the message without its `(max allowed 2.00 cm)` tail, which is the part that shows the comparison ran against the 2 cm default. **(v) The generalising rule behind the table.** The actors that fail are exactly the actors whose mesh has geometry overhanging its own footprint; nothing else separates the passing rows from the failing ones. **(vi) Measurement environment.** Host project EAContentExamples58 (map as forcing function), `/Game/Maps/Atlantis`, UE 5.8, PinWright at that checkout's HEAD. **(vii) Severity qualifiers.** The duplicate's rationale agreed `High` and added two bounds this ticket did not state: the defect **does not take the editor down** (so not `Critical`), and the workaround exists but consists of discarding the verb's headline output and re-deriving the verdict from `coverage`/`contactPoints`/`groundSpreadCm` by hand — the manual box-math the verb was written to replace. **Tags merged** from the duplicate: `max-gap`, `footprint-columns`, `overhang`, `curvature`, `misleading-failreason`. Its hyphenated spellings of tags this ticket already carries (`verify-grounding`, `ground-actors`, `false-failure`) were deliberately not added — they are synonyms of `verify_grounding` / `ground_actors` / `false-negative` and would only fragment tag search.
- `#3-gate-on-the-seating-scoped-gap` `IN-REVIEW` developer — Suggested fix #1 implemented as a decomposition rather than a column subset. A column's clearance is `(UndersideZ_i - min_j UndersideZ_j) + (min_j UndersideZ_j - GroundZ_i)`: the first term is the actor's own underside RELIEF (pure mesh shape, present however well it is seated), the second is the FLOAT (placement). `FGroundContactReport::MaxGapCm` — the field `maxGap` bounds and the only one that gates `bPass` — is now the float term at its worst, `min(UndersideZ) - min(GroundZ)`, computed in a new `GroundPlacement::AggregateColumns` split out of `MeasureContact` (`Handlers/Spatial/GroundPlacementUtils.cpp`). Two new reported-never-gated fields carry the rest of the measurement, per this ticket's own "report both" wording: `MaxColumnClearanceCm` (the old silhouette max, serialized as `maxColumnClearanceCm`) and `UndersideReliefCm` (`undersideReliefCm`, max minus min underside sample — the size of the shape term). **The new rule is provably weaker than the old one:** `MinGapCm <= MaxGapCm <= MaxColumnClearanceCm` for every possible sample set, so nothing that passed the silhouette gate can start failing; a test asserts the chain on every fixture. On the reported repro the intact column's gate figure becomes `-48.43` (PASS) with `maxColumnClearanceCm: 2327.90` still visible beside it, and the toppled shaft on one end still fails because its lowest point really does sit above the lowest ground under it. `misleadingFailReason` addressed at the same site (`GroundPlacementUtils.cpp` `EvaluateContact`): the message now names the compared quantity and says outright that overhang is not counted — "This actor's lowest point sits %.2f cm above the lowest ground under its footprint (max allowed %.2f cm)… its underside varies by %.2f cm, largest single-column clearance %.2f cm." The `maxGap` param description and the verb summary in `Handlers/Spatial/GroundPlacementHandler.cpp` were rewritten to match; the shared `GroundRpcContactObject` gained the two additive fields, so `spatial.ground_actors` rows carry them too and the two verbs no longer disagree about what `maxGapCm` means. One line outside the two grounding verbs: `Handlers/Level/LevelAuditUtils.cpp:343` now copies `Report.MaxColumnClearanceCm` into its own `FGroundMeasure::MaxGapCm`, so `level.audit`'s two findings that echo that number keep reporting the silhouette figure they have always reported (it gates on `MinGapCm` and sets `bEnforceGapBounds=false`, so no audit verdict moves). Tests added to `Tests/Spatial/TestGroundPlacement.cpp`: `PinWright.spatial.verify_grounding.OverhangIsNotFloating` (constructed columns, flat ground in every fixture so the mesh is the only variable — a capital-over-a-round-base column bedded 48 cm PASSES where it used to fail on a 2352 cm "float", a cylinder on its flank bedded 22 cm PASSES where it used to fail on 60.9, a 500 cm floating box still FAILS, and a flat slab perched on one high point over a 300 cm drop still FAILS through the gap bound itself rather than the zero-contact branch) and `PinWright.spatial.verify_grounding.CurvedUndersideStillPasses` (engine sphere resting tangent on a cube floor, through the real RPC: `pass:true`, `maxGapCm < 2`, `maxColumnClearanceCm`/`undersideReliefCm` both > 10 so the row can still be judged by hand). Not done, and split out rather than smuggled in: the `HOLDER_NOT_SEATABLE` refusal for ISM/HISM holders this ticket also raises — it belongs on BOTH grounding verbs and `spatial.ground_actors` was another agent's scope this wave; filed as a follow-up in the fix report. Not compiled or run — this wave's agents do not build.
- `#4-verified-fixed` `DONE` verifier — 2026-08-28. The ticket's own repro re-run live on the editor
  rebuilt at `b79ba53e`, against the same thirteen `AVE_Col_*` actors still standing on
  `/Game/Maps/Atlantis` (`actor.list {filter:"AVE_Col"}` -> 13, `StaticMeshActor_3..15`). Verbatim
  call `spatial.verify_grounding {surface:{preset:"landscape"}, prefix:"AVE_Col_", samples:4,
  maxPenetration:60, detail:"all"}` -> **`passed: 12, failed: 1`**, against the filed
  `passed: 5, failed: 8`. All four intact Doric columns flipped, and the silhouette number survives
  beside the gate figure exactly as `#3` promised: `AVE_Col_N1_intact` `pass:true`,
  `maxGapCm: -48.43`, `maxColumnClearanceCm: 2327.897`, `undersideReliefCm: 2376.33`; `N4_intact`
  `-48.47` / `2269.23`; `S0_intact` `-48.45` / `2272.76`; `N6_intact` `-48.40` / `2266.82` — i.e. the
  gate now reads the exact numbers the ticket's table called `minGapCm`, and the exact numbers it
  called `maxGapCm` are still reported, renamed. The six rows that already passed (`N3_snapped`,
  `S1_snapped`, `S4_snapped`, `N0_stump`, `N2_stump`, `S2_stump`) all still pass, so nothing was
  loosened out from under them.
  **The gap gate is still live, not switched off** — checked with a threshold differential rather
  than inferred, since the one remaining failure (`AVE_Col_N2_shaft`, `contactPoints: 0`,
  `maxGapCm: 3.31`) exits through the zero-contact branch and would not have exercised the bound.
  Same two actors, only `maxGap` changed from 2 to 1:
  `spatial.verify_grounding {actors:["AVE_Col_S3_shaft","AVE_Col_N1_intact"], samples:4,
  maxPenetration:60, maxGap:1, detail:"all"}` -> `S3_shaft` `pass:false`, `ACTOR_NOT_GROUNDED`
  through the bound itself (`contactPoints: 1`, so not the zero-contact path) with the reworded
  reason: *"This actor's lowest point sits 1.48 cm above the lowest ground under its footprint (max
  allowed 1.00 cm), so nothing is holding it up. Measured from the actor's own lowest underside
  sample: geometry that overhangs or curves away higher up is shape, not float, and is not counted
  here (its underside varies by 219.95 cm, largest single-column clearance 221.43 cm)."* — which is
  suggested fix #3's ask, satisfied. In the same response and at the same tightened threshold
  `N1_intact` still **passes** on `maxGapCm: -48.43` while carrying `maxColumnClearanceCm: 2327.90`,
  so a 2327 cm capital no longer votes on float at any threshold that still rejects a 1.48 cm one.
  That is the separation the ticket said no value of `maxGap` could buy under the old rule.
  Not verified here and deliberately left out of scope: `HOLDER_NOT_SEATABLE` for ISM/HISM holders,
  which `#3` split out.
