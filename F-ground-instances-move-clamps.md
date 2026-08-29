---
id: F-ground-instances-move-clamps
title: "`spatial.ground_instances` has no parameter that bounds or refuses a move by its magnitude or sign, no revert on a failed seat, and no per-instance delta in a successful applying response — 38 of 38 spilled applying payloads carried `movedInstances[]` of `{index, previousTransform}` and nothing else — so 'never lift an already-bedded prop' and 'never bury a log that was lying correctly' can only be enforced by spending a whole extra dry-run measurement pass on a verb that is batch-only precisely because of game-thread cost"
status: OPEN
severity: Medium
category: feature
tags: [spatial, ground_instances, ism, hism, instanced-static-mesh, scatter, vegetation, apply, dry-run, clamp, max-lift, max-sink, delta, movedinstances, detail, undo, revert, game-thread-cost, placement, level-building]
encounters: 1
lastSeen: 2026-08-29T20:50:00+03:00
---

# The verb will move an instance any distance in either direction, and there is no way to say "not that far"

`spatial.ground_instances` writes. Its seventeen parameters tune *how the seat is measured* — grid
size, footprint inset, percentile, embed, coverage and contact-point thresholds, a readback
tolerance. **Not one of them says anything about the resulting move.** There is no `maxLift`, no
`maxSink`, no `maxDeltaZ`, and no refusal keyed on the magnitude or sign of `DeltaZ`. The solve
computes it and applies it in the next two lines.

That is fine while the solve is right. Two rules that any bulk re-seat needs — **never lift something
that is already bedded** and **never bury something that is already lying correctly** — are not
expressible, and both were violated in one pass on this level, each with `pass: true` on the row.

## What the applying response gives you instead

**Correction to the report this ticket was filed from, stated first because it changes the ask.**
The applying path *does* compute and publish a per-instance `deltaZCm` — the claim that it does not
is wrong:

    GroundPlacementHandler.cpp:1579-1581   if (Result.Seat.WasMoved())
                                           {
                                               Row->SetNumberField(TEXT("deltaZCm"), Result.Seat.AppliedDeltaZCm);

The defect is *where* it lives. That row goes into `results[]`, which is gated by `detail`
(`GroundPlacementHandler.cpp:1352-1356`, default **`"failures"`**) through the selector at `:1546`
(`const bool bWantRow = bPlaced ? bIncludeSuccesses : bIncludeFailures;`) and capped at 256 rows
(`GroundRpcMaxDetailRows`, `:64`, enforced `:1551-1555`). **On a fully successful apply there are no
rows at all.** What is ungated is `movedInstances[]` (`:1537-1544`, attached `:1639`), and it carries
exactly two fields:

    GroundPlacementHandler.cpp:1540      Undo->SetNumberField(TEXT("index"), Result.InstanceIndex);
    GroundPlacementHandler.cpp:1541-1542 Undo->SetObjectField(TEXT("previousTransform"),
                                             GroundRpcTransformObject(...PreviousTransform.GetValue()));

Measured over this session's spilled payloads (`Saved/PinWright/HttpResponses/`): **38 applying
`ground_instances` responses, 7,920 instances moved, largest single batch 630 — and exactly one of
the 38 carried a `results[]` array at all.** That one is the only batch with `failed > 0` (42 of 630
on `HISM_ZF_Bramble`). Every other apply, including three that each moved 268-324 instances,
published index-and-previous-transform and nothing else. The verb's own summary sentence is accurate
about this and reads as reassurance: *"Every instance this verb moves is listed in movedInstances[]
with its pre-move transform, at every detail level - that is the undo"* (`:1257-1259`). It is the
undo. It is not a report of what happened.

**And there is no revert.** `GroundPlacementUtils.cpp:1598-1601`:

> `// No revert branch. The instance stays where the solve put it and CurrentTransform /`
> `// PreviousTransform carry what it was, which actor.set_instance_transforms can write back`
> `// verbatim`

`FGroundSeatConfig::bRevertOnFailure` exists (`GroundPlacementUtils.h:556`) and is never wired for
instances. An instance whose post-move readback fails is left moved. Combined with
`B-foliage-mutators-no-transaction` (no `FScopedTransaction` on the foliage mutators), the response
body is the only record that the move happened — so a caller who discards it, or whose client drops
a 250 KB payload, has no undo at all.

## The two rules, and the damage each one caught

Both measured on `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8) during a
2,293-instance re-seat across 28 components. Method in `Docs/map/tree-seating-on-slopes.md` § *Two
acceptance rules, both of which caught real damage*; producers under `dev/planting/`.

**1. Never lift.** After the first apply, **105 instances measured worse than before**; **38** were
handed back from a pre-recorded original set (count re-derived from the five revert payloads still on
disk in `dev/planting/out/args/` — 19 + 6 + 4 + 2 + 7). Root cause is filed separately as
`B-ground-instances-rotated-aabb-underside-plane`: below roughly `seatPercentile 0.75` the seat lifts
an already-bedded rotated prop, because the plane it seats to is the rotation-inflated AABB minimum.
A `maxLift: 0` would have refused every one of those 38 without needing that diagnosis at all.

**2. Never over-bury, and this half has nothing to do with rotation.** `seatPercentile: 1` is the
parameter at its documented maximum doing exactly what it says (*"1 sinks until no column floats"*,
`GroundPlacementHandler.cpp:1318-1322`). On `InstancedFoliageActor_0` /
`FoliageInstancedStaticMeshComponent_51`, 17 `SM_Driftwood`, with `samples: 7, embedDepth: 8,
embedFraction: 0`, the dry run proposed sinks on 10 of 17 and a **minimum `proposedDeltaZCm` of
-271.4 cm** — one log driven 2.7 m further into the terrain, taking it from ~1.3 m below the surface
to ~4.0 m below it. **`pass: true` on every one of the 17 rows.** The mesh-underside metric
(`dev/planting/p_gap_mesh.py`, lowest LOD0 vertex per cell of a 4x4 local-XY grid) says **0 of the 17
had whole-object float before the pass**, so none of those ten needed moving at all. All 10 were
reverted, plus 1 in `...Component_28`. A capture confirmed both states: before, the log lay on the
surface; after, only a fragment showed above the grass.

Nothing in the verb can express either rule. The refusal parameters it has are all about
*measurement quality*, not move size — `minCoverage`, `minContactPoints`, `contactTolerance`
(`:1330-1341`) and `maxSeatError` (`:1342-1346`, a bound on predicted-vs-measured *disagreement*,
not on distance travelled). The two thresholds that are absolute distances, `MaxGapCm` and
`MaxPenetrationCm`, are **deliberately switched off on this path** — `PreThresholds.bEnforceGapBounds
= false` (`GroundPlacementUtils.cpp:1465`) and again post-move (`:1558`) — for a stated and correct
reason (`GroundPlacementUtils.h:289-297`: an absolute gap bound would fail a legitimate first-contact
rest on a slope). That reasoning is about the *gap*; it says nothing about the *move*, which is why a
move bound does not reopen it.

## Why the workaround is not free

The workaround exists and works: dry run, read `proposedDeltaZCm` (`:1593`), filter, then apply with
`indices` (`:1287-1289`). It is what this project did. Three costs, in order of size.

**Game-thread trace work.** The verb's own `samples` documentation states the unit
(`GroundPlacementHandler.cpp:1310-1312`): *"Cost is samples^2 columns per instance per measurement,
and each seated instance is measured twice."* An apply is two measurements per instance; a dry run is
one; so dry-run-then-apply over the same set is **1.5x** the tracing of applying alone — *not* the
2x the original report claimed, and worth stating precisely because the argument is a cost argument.
At `samples: 5` over this level's 2,293 instances that is roughly 57,000 extra columns of complex
tracing to establish a rule the verb could have enforced with one comparison per instance. And this
verb is batch-only *because* of exactly this: `GroundPlacementHandler.cpp:8-10` — *"A per-actor Python
loop over a level is what wedged an editor for 168 minutes with 5100 uncancellable calls"* — with the
batch ceiling *"chosen to bound worst-case game thread time rather than response size"* (`:54-58`).
A verb whose whole shape is a cost decision is a strange place to make correctness cost 50% more.

**A second full response.** Each dry run at `detail: "failures"` emits a row per instance, *"which on
a dry run is every one of them"* (`:1353-1354`), each carrying a full `contact` object with
`groundProvenance`. The dry runs behind the table above ran 176-921 KB each.

**A round trip with its own hazard.** Instance indices are positional, and an out-of-range index is a
hard refusal rather than a skip (`:1440-1462`, `ERR_INSTANCE_INDEX_OUT_OF_RANGE`, *"NOTHING WAS
MOVED"*) — correct behaviour, and it means the filtered apply must also carry `expectedCount`
(`:1290-1298`) to be safe. That guard is not theoretical here: during this pass PCG renamed
`ISM_HillTree_P2_23/24/25` to `_0/_1/_2` between two calls in the same session, which is why every
write was preceded by a component re-resolve (`dev/planting/p_resolve.py`).

## Ask

**1. `maxLift` and `maxSink` (cm, absolute, each optional; omitted means unbounded).** When the
solved `|DeltaZ|` exceeds the bound in that direction, **refuse the instance** — leave it untouched,
do not clamp-and-move — and emit it as a failure row with a distinct `reasonCode` (e.g.
`SEAT_MOVE_EXCEEDS_BOUND`) carrying the proposed delta, so the caller learns which instances the rule
caught without a second call. Refusal rather than clamping matters: a clamped move is a seat that
satisfies nothing, and it would arrive indistinguishable from a good one. `maxLift: 0` is then the
whole of "never lift", expressible in one parameter.

The insertion point is one comparison. `GroundPlacementUtils.cpp:1510` computes `DeltaZ` and
`:1512-1513` proposes the transform from it; the dry-run and applying branches both flow from there.

**2. Put the delta where a successful apply publishes it.** Either add a compact `deltaZCm` to each
`movedInstances[]` row — it is already the ungated mutation receipt, and one number per row is small
beside the transform already there — or publish batch-level move statistics beside `moved` / `placed`
(count lifted, count sunk, min / median / max `deltaZCm`). The second is cheaper and would have made
both defects above visible in the response that caused them: "moved 324, lifted 245" is a sentence a
caller acts on.

Either half is useful without the other. (2) alone makes the damage visible after the fact; (1) alone
prevents it. (1) is the one this ticket is named for.

## Not a duplicate of

- **`E-ground-provenance-unreachable-at-summary-detail`** (OPEN, Medium) — structurally the same
  shape (a field attached only to a `detail`-gated row, therefore absent from a clean batch) on a
  different field, and the two do not fix each other: that ticket's ask is a **batch-level contact
  object**, which by construction cannot carry a per-instance delta, and this ticket's ask is a
  **bound applied before the write**, which no amount of provenance reporting supplies. Worth reading
  together — a fixer touching the row/echo split should serve both.
- **`F-ground-instances-failure-rows-need-a-non-applying-seat`** (OPEN, Medium) — that a *dry run*
  cannot select genuine failures because every dry-run row is not-placed by construction. This ticket
  is about the *applying* run and about a bound that does not exist at either setting. Its fix (a
  non-mutating seat that reports real failures) would make the dry-run half of this workaround
  cheaper to read; it would not remove the extra measurement pass and would not bound anything.
- **`E-ground-instances-two-truncation-flags`** (OPEN, Low) — the row cap announcing itself under an
  undocumented name. The cap is cited here only to show that even at `detail: "all"` a 324-instance
  apply cannot report a delta for every instance it moved.
- **`B-ground-instances-rotated-aabb-underside-plane`** (OPEN, Medium) — filed from the same pass, and
  the cause of rule 1's damage. It asks for a way to *understand* the move; this asks for a way to
  *bound* it. They mitigate each other and neither is the other's fix: a `maxLift` refuses the bad
  write without explaining it, and the disclosure explains it without preventing it. Rule 2 above is
  evidence this ticket needs independently of that one — an over-bury at `seatPercentile: 1` involves
  no rotation inflation whatsoever.
- **`B-foliage-mutators-no-transaction`** (OPEN) — no editor transaction on the foliage mutators.
  Cited above as what makes the receipt load-bearing rather than convenient; not re-filed here.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* carries the enumeration and it is not
restated here. This ticket sits beside that class rather than inside it: the response is not wrong
and no reported number is misleading — the deciding number is computed, is correct, and is simply
not on the object a successful apply returns. The nearest board sibling in *that* precise shape is
`E-ground-provenance-unreachable-at-summary-detail`, which is the same `detail`-gated-row problem on
a different field, and the two should be read together by anyone touching the row/echo split.

Its true partner on this board is `B-ground-instances-rotated-aabb-underside-plane`, filed from the
same pass: that ticket is why the solve was wrong, this one is why nothing stopped the wrong solve
from being written. A board that fixes only the first still has no rail for the next wrong solve, and
a board that fixes only this one still needs a dry run to choose the bound.

## Severity

**Medium**, on the rubric's soft-blocker band: *"doable, but only via a documented workaround, a
source dive, or many extra calls"*. All three clauses are literally true — `indices` and `apply:false`
are documented, the cost accounting above needed the source, and the extra calls are a full second
measurement pass over the whole selection.

**Not High.** The High bands are silent wrong data and a hard blocker with no workaround. Neither
holds: the applying response is truthful about what it did (`movedInstances[]` is complete and exact,
and `moved` / `placed` / `failed` are counted, never asserted — `GroundPlacementHandler.cpp:1562-1563`
derives both booleans rather than writing literals, and says so in a comment at `:1559-1561`), and the dry-run route
genuinely works. What is missing is a safety rail, not a truth.

**Not Critical, and the case against is worth stating** because this ticket is about a write that
damaged content. Critical is *"a write that corrupts or loses asset data"*. Nothing was lost: every
instance moved in this pass was recoverable, and was in fact recovered, from transforms the caller
held. But that recovery depended on the caller having recorded originals **out of band** before the
first call — `dev/planting/out/undo/originals_bulk.json`, written by `p_bulk_prep.py` from
`get_instance_transform`, not from any response. A caller who trusted `movedInstances[]` alone would
have been fine too, provided they kept all 38 responses. It stays Medium; but a fix that adds
`bRevertOnFailure` for instances, or that lands the move bound, is also the thing standing between
this and a real data-loss report.

**Reach modifier declined, and the argument for it is the stronger one this time.**
`spatial.ground_instances` is the plugin's only per-instance grounding verb, `HOLDER_NOT_SEATABLE`
(`GroundPlacementHandler.cpp:718` on `ground_actors`, `:1038` on `verify_grounding`) routes every
ISM/HISM caller here by construction, and unlike `B-ground-instances-rotated-aabb-underside-plane`
this gap has **no mesh-class or rotation restriction at all** — it applies to every instance every
call makes. That is a genuine case for High. It is declined because reach in the rubric is about how
often the *method* runs, and this is a level-building verb rather than an every-session one; and
because the absent capability is a guard rail on a path that is already correct when the caller checks
first. Medium, worked before the Lows, is the right placement.

## History
- `#1-no-way-to-bound-the-move` `OPEN` reporter — Filed from a 2,293-instance bulk re-seat across 28
  components on `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8); method in
  `Docs/map/tree-seating-on-slopes.md` § *Bulk re-seat*, producers under `dev/planting/`. Mechanism
  re-derived at PinWright HEAD `0f4f9594`. **Two premises from the report this was filed from were
  wrong and are corrected in the body rather than carried.** (1) "The applying path returns no
  `proposedDeltaZCm`" — it returns `deltaZCm` (`GroundPlacementHandler.cpp:1581`, gated `:1579` on
  `WasMoved()`), but only inside `results[]`, which the `detail` default of `"failures"`
  (`:1352-1356`) empties on a successful batch; the ungated `movedInstances[]` (`:1537-1544`) carries
  only `index` and `previousTransform` (`:1540`, `:1541-1542`). Re-derived by census over this
  session's spilled payloads: 38 applying responses, 7,920 instances moved, exactly **one** carrying
  a `results[]` array, and that one is the only batch with a failure. (2) "at double the game-thread
  cost" — an apply is two footprint measurements per instance and a dry run is one (`:1310-1312`), so
  the dry-run-then-apply route is **1.5x**, not 2x; the cost argument is stated at the corrected
  figure. Full 17-parameter ParamSpec block (`:1262-1357`) read in full: nothing keyed on the
  magnitude or sign of the move; `minCoverage` / `minContactPoints` / `contactTolerance` /
  `maxSeatError` are measurement-quality gates, and the two absolute-distance thresholds are switched
  off on this path on purpose (`GroundPlacementUtils.cpp:1465`, `:1558`, rationale
  `GroundPlacementUtils.h:289-297`). No revert either: `GroundPlacementUtils.cpp:1598-1601`, with
  `bRevertOnFailure` (`GroundPlacementUtils.h:556`) never wired for instances. Two independent classes
  of measured damage, both with `pass: true` on every row and only one of them explained by
  `B-ground-instances-rotated-aabb-underside-plane`: 105 instances worse after the first apply with 38
  reverted (count re-derived from the five payloads in `dev/planting/out/args/`), and a
  `seatPercentile: 1` over-bury that proposed sinks on 10 of 17 driftwood at a minimum
  `proposedDeltaZCm` of **-271.4 cm**, on logs the mesh-underside metric says were 0 of 17 floating.
  Ask is `maxLift` / `maxSink` that **refuse** rather than clamp, inserted at the one comparison
  around `GroundPlacementUtils.cpp:1510`, plus a delta on the ungated receipt or batch-level move
  statistics. Severity Medium on the soft-blocker band; High declined because the response is truthful
  and the dry-run workaround genuinely works; Critical declined with the argument stated, since
  recovery in this pass depended on originals recorded out of band; reach bump declined although this
  gap, unlike its sibling, has no mesh-class or rotation restriction.
