---
id: F-ground-instances-move-clamps
title: "`spatial.ground_instances` cannot bound or refuse a move by its size or sign (no `maxLift` / `maxSink`), and a successful apply never reports the per-instance `deltaZCm` — `movedInstances[]` carries only index, previous transform and space"
status: OPEN
severity: Medium
category: feature
tags: [spatial, ground_instances, ism, hism, instanced-static-mesh, scatter, vegetation, apply, dry-run, clamp, max-lift, max-sink, delta, movedinstances, detail, undo, revert, game-thread-cost, placement, level-building]
encounters: 1
costly: 1
lastSeen: 2026-08-29T20:50:00+03:00
rice: [1, 3, 1, 1]
priority: 33
---

# `spatial.ground_instances` has no move bound, and a clean apply does not say how far anything moved

`spatial.ground_instances` writes, and every refusal parameter it has is about *measurement
quality* — `minCoverage`, `minContactPoints`, `contactTolerance`, `maxSeatError`
(`Source/PinWright/Private/Handlers/Spatial/GroundPlacementHandler.cpp:1372-1388`). None is keyed on
the resulting move. The solve computes `DeltaZ = -(SeatClearance + EmbedCm)` and applies it
(`Source/PinWright/Private/Handlers/Spatial/GroundPlacementUtils.cpp:1720-1724`) whatever its
magnitude or sign, so two rules every bulk re-seat needs — **never lift an already-bedded instance**
and **never bury one that was lying correctly** — cannot be expressed.

Both were violated in one measured pass, each with `pass: true` on the row (host
`EAContentExamples58`, UE 5.8, `/Game/Maps/PW_VegetationTest`, 2,293 instances over 28 components):
105 instances measured worse after the first apply and 38 had to be reverted (the lift cause is
`B-ground-instances-rotated-aabb-underside-plane`); and `seatPercentile: 1` (*"1 sinks until no
column floats"*, `GroundPlacementHandler.cpp:1360-1364`) on 17 `SM_Driftwood` proposed sinks on 10 of
17 with a minimum `proposedDeltaZCm` of **-271.4 cm**, on logs a mesh-underside metric measured as 0 of
17 floating. The second case involves no rotation inflation; it is the parameter at its documented
maximum.

A successful apply also does not report the move. `deltaZCm` is written only on `results[]` rows
(`GroundPlacementHandler.cpp:1643-1645`), which `detail` gates (default `"failures"`, selector
`:1610`) and caps at 256 (`GroundRpcMaxDetailRows`, `:65`), so a clean batch has no rows. The ungated
receipt `movedInstances[]` is built by `InstancedMeshUtils::MakeMovedInstanceRow`
(`GroundPlacementHandler.cpp:1597-1607`, `Source/PinWright/Private/Handlers/Actor/InstancedMeshUtils.h:277`)
with index, `previousTransform` and `space` only. Across 38 spilled applying responses (7,920
instances moved) exactly one carried `results[]`, the only one with `failed > 0`.

Undo itself is covered: the apply path runs inside one `FScopedTransaction`
(`GroundPlacementHandler.cpp:1570-1580`, documented at `:1281`), so `editor.undo` reverts a whole
batch. That reverts everything, not just the instances a rule would have refused.

**Workaround:** dry run (`apply: false`), read each row's `proposedDeltaZCm`, filter, then apply with
`indices` plus `expectedCount` (`GroundPlacementHandler.cpp:1310-1321`). Each seated instance is
measured twice and a dry-run instance once (`samples` doc, `:1331-1335`), so this costs 1.5x the
tracing of a plain apply plus a second full response (176-921 KB per dry run in the measured pass).

**Fix:**
1. Add optional `maxLift` and `maxSink` (cm, absolute; omitted = unbounded). When the solved `DeltaZ`
   exceeds the bound in that direction, **refuse the instance** (no write, no clamp) and report it as
   a failure row with a distinct `reasonCode` (e.g. `SEAT_MOVE_EXCEEDS_BOUND`) carrying the proposed
   delta. The check sits right after `DeltaZ` is computed (`GroundPlacementUtils.cpp:1721`), so the
   dry run and the apply both honour it. Refuse, not clamp: a clamped seat satisfies nothing and would
   look like a good one.
2. Add `deltaZCm` to each `movedInstances[]` row (or batch-level lifted/sunk counts and min/median/max
   `deltaZCm` beside `moved`), so a clean apply states how far it moved things.

**Acceptance:** `maxLift: 0` on a batch whose solve lifts some instances leaves those instances'
transforms unchanged, counts them in `failed`, and emits rows with the new `reasonCode` and the
proposed delta; `maxSink: 50` does the same for any proposed sink deeper than 50 cm; with neither
passed, behaviour is unchanged; a fully successful apply at `detail: "summary"` exposes each moved
instance's `deltaZCm` (or the batch move statistics).

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
- `#2-rephrased` `OPEN` developer — Re-checked at PinWright `7230b41d`. All line citations were stale (shifted ~+60) and are refreshed. Dropped the claim that the response body is the only undo and the `B-foliage-mutators-no-transaction` dependency: the apply path now runs inside one editor transaction (`GroundPlacementHandler.cpp:1570-1580`, summary `:1281`), so `editor.undo` reverts a batch. Dropped the per-instance revert ask (`GroundPlacementUtils.cpp:1812` still has no revert branch, but transaction undo covers recovery). Still real and kept as the ask: no `maxLift` / `maxSink` refusal, and `movedInstances[]` (`InstancedMeshUtils::MakeMovedInstanceRow`, `GroundPlacementHandler.cpp:1606`) carries no `deltaZCm`. Body trimmed to the template with Fix and Acceptance. Severity unchanged (Medium).
