---
id: B-ground-instances-single-sample-degenerate
title: "`samples: 1` is a legal value that silently disables every acceptance check the grounding verbs offer — `coverage` becomes exactly 1.0, `groundSpreadCm` and `undersideReliefCm` become exactly 0, and `contactPoints` becomes 1 against a default `minContactPoints` of 1, all by construction — and the single column does not land where the word 'centre' in the docs leads a caller to think: measured p50 152.7 cm and max 324.6 cm from the instance pivot, because it lands at the world AABB centre after yaw inflation"
status: OPEN
severity: Medium
category: bug
tags: [spatial, ground_instances, ground_actors, verify_grounding, samples, grid-size, single-point-rest, coverage, contact-points, ground-spread, vacuous-check, degenerate, bounds-centre, aabb, docs, placement, level-building, vegetation]
encounters: 1
lastSeen: 2026-08-29T20:10:00+03:00
---

# One column is not a footprint, and it is not under the object either

`samples` is clamped to 1-9 and 1 is legal. At 1 the grounding verbs still return a full contact
report, still compute `pass`, and still honour `minCoverage` / `minContactPoints` — but every one of
those numbers is fixed by the parameter rather than by the seat. Nothing in the response says so.

Separately and worse, the one column is not under the object's pivot. Both the parameter description
and the wiki call it "a single centre column"; the centre in question is the **world-space AABB
centre after the instance's rotation has been re-fitted to an axis-aligned box**, which on a measured
177-instance tree component sits a median of **152.7 cm** and a maximum of **324.6 cm** away from the
pivot — well outside the mesh's 100 uu contact radius.

## Mechanism, re-derived at HEAD `6d0e91a3`

All paths in `Plugins/PinWright/Source/PinWright/Private/Handlers/Spatial/`.

**1 is legal, and 0 or negative silently becomes 1.**

    GroundPlacementHandler.cpp:1419-1423
        Config.GridSize = FMath::Clamp(
            Ctx.GetInt(TEXT("samples"),
                Ctx.GetInt(TEXT("gridSize"),
                    Ctx.GetInt(TEXT("grid_size"), GroundPlacement::DefaultGridSize))),
            GroundPlacement::MinGridSize, GroundPlacement::MaxGridSize);

`MinGridSize = 1` (`GroundPlacementUtils.h:61`), `DefaultGridSize = 3` (`:60`). **There is no refusal,
no warning and no response flag anywhere in the plugin** — `GridSize == 1`, `Samples == 1` and
`== MinGridSize` have zero occurrences across the whole source tree. The value is echoed back at
`GroundPlacementHandler.cpp:1564` (`SeatEcho->SetNumberField(TEXT("samples"), Config.GridSize);`) and
that echo is the caller's only signal, which requires already knowing what it means.

**The one column lands at the AABB centre.**

    GroundPlacementUtils.cpp:974-982
        for (int32 IX = 0; IX < Grid; ++IX)
        {
            // A 1x1 grid samples the centre; larger grids span [-half, +half] inclusive.
            const double FracX = (Grid == 1) ? 0.5 : (static_cast<double>(IX) / static_cast<double>(Grid - 1));
            const double X = Origin.X - HalfX + 2.0 * HalfX * FracX;
            for (int32 IY = 0; IY < Grid; ++IY)
            {
                const double FracY = (Grid == 1) ? 0.5 : (static_cast<double>(IY) / static_cast<double>(Grid - 1));
                const double Y = Origin.Y - HalfY + 2.0 * HalfY * FracY;

With `Grid == 1` this reduces to `X = Origin.X`, `Y = Origin.Y`, and `Origin` is
`WorldBounds.GetCenter()` (`:922`). Two consequences a caller has no way to anticipate:

- **`footprintInset` becomes inert.** `HalfX`/`HalfY` (`:934-935`) cancel out of the expression
  entirely at `Grid == 1`. A caller tuning inset alongside `samples: 1` is turning a dead knob.
- **The offset is per-instance and depends on yaw.** The box handed in is
  `Mesh->GetBounds().GetBox().TransformBy(InstanceWorld)` (`:1450`), and `FBox::TransformBy` returns
  the axis-aligned hull of the *rotated* box. So the centre moves with the instance's rotation as
  well as with the mesh's own bounds origin.

**Every acceptance field collapses.** In the `BoundsPlane` branch used by `ground_instances`,
`bHasActorGeometry` is set unconditionally (`:998`), so the single column is always tallied
(`:505-508`, `++Report.ActorColumns`) and `ActorColumns == 1`. Then:

- `coverage` — `:611-613`, `SupportedColumns / ActorColumns`. At a denominator of 1 the value is
  binary, and the `0` case never reaches the coverage gate because `EvaluateContact` exits earlier
  with `GROUND_NOT_FOUND` when nothing is supported (`:725-731`). **So among rows that reach the
  check, `coverage` is exactly 1.0, always.** `minCoverage` is clamped to 0.0-1.0 at all three read
  sites, so no threshold can reject it.
- `groundSpreadCm` — `:580`, `MaxGround - MinGround`. Both are seeded from the single supported
  column at `:549-550` and only widened in the second-column branch at `:559-560`, which never runs.
  **Identically 0.**
- `undersideReliefCm` — `:578`, same degeneracy; and on `ground_instances` it is 0 anyway because the
  underside model is a flat plane (`B-ground-instances-footprint-is-bounds-not-contact`).
- `contactPoints` — 1 at most, against a `minContactPoints` that defaults to **1** at every read
  site. The only absolute guard is satisfied by the parameter that broke the measurement.

`maxGapCm`, `minGapCm` and `maxColumnClearanceCm` also collapse onto one value for the same reason.

## Measured

Host `EAContentExamples58`, UE 5.8, `/Game/Maps/PW_VegetationTest`; method and numbers in
`Docs/map/tree-seating-on-slopes.md`, producers under `dev/planting/`.

**Where the column actually lands** (`dev/planting/p_column.py`, all 177 instances of
`Actor_3 / HISM_ZF_HillTree`): the ground height the seat solve implies matches ground probed at the
**AABB centre to within 0.01 cm**, and differs from ground probed at the **pivot** by rms 39.2 cm,
absolute max 172.7 cm. Pivot-to-AABB-centre distance is **p50 152.7 cm, max 324.6 cm**. `HillTree_P2`
has a bounds origin 167.8 uu off the trunk axis in Y before any yaw is applied, and its contact
radius is 100 uu — so at `samples: 1` the verb is measuring ground the object does not stand on, on
every instance.

**What the seat reports at that setting** — `spatial.ground_instances {apply:false, samples:1}` over
the same 177 instances: `actorColumns` 1, `coverage` **1.000**, `groundSpreadCm` **0** for every
single row, `pass: true` throughout, while proposing dZ p50 +41.2 / p90 +89.3 / max +157.4 cm on
instances that were already bedded.

**What it produces in the field.** The one component in this level that was seated with `samples: 1`
(`InstancedFoliageActor_0 / FoliageInstancedStaticMeshComponent_1`, 208 instances) is measurably the
worst of the six tree components and the **only one with a positive median float**: float p50 +14.8,
gap p50 +24.3, p90 +104.4, max +252.1 cm. The five components seated at the default `samples: 3` run
gap p50 -35.6 to -13.0. It passed every check the verb offers.

## Premise correction — the docs do not recommend this, and that is worth stating

The report this ticket was filed from claimed `samples: 1` is "currently recommended practice in the
wiki". **It is not**, and the wiki was checked rather than assumed:

- `Plugins/PinWright/docs/wiki-src/spatial.ground-placement.md:73` — *"Lower `samples` to 1 only if
  you want the old single-point behaviour back."*
- `Plugins/PinWright/docs/wiki-src/spatial.md:273` — *"`1` is the old single-point behaviour."*
- `GroundPlacementHandler.cpp:784-785` (`ground_actors` ParamSpec) is the most pointed of the three —
  *"1 degrades to a single centre column (the old pivot-probe behaviour, and the reason a boulder
  ends up balanced on one point)."*

The recommendation existed only in the host project's own working notes, and has since been
superseded there (`Docs/map/tree-seating-on-slopes.md`). **What survives is narrower and still real:**

1. Every one of those three lines says *"centre"* or *"pivot-probe"* without qualification, and the
   `ground_instances` ParamSpec (`GroundPlacementHandler.cpp:1282`) drops even the boulder warning:
   *"1 degrades to a single centre column."* A caller reads "centre" as the object's centre or its
   pivot. It is the world AABB centre after yaw inflation, measured 152.7 cm away at p50. **"The old
   pivot-probe behaviour" is the single most misleading phrase available**, because the one thing the
   column is provably not is a pivot probe (rms 39.2 cm disagreement, above).
2. None of the three says that the acceptance fields stop measuring anything. A caller who reads the
   caution as "the seat will be less accurate" and keeps `minCoverage` / `minContactPoints` as their
   guard has no guard at all, and nothing tells them.

## Ask

Not a refusal — `samples: 1` is a legitimate cheap mode for a flat-bottomed prop on flat ground, and
refusing it would break callers. **Make it impossible to use it without knowing.** In order of value:

1. **A `warnings[]` entry on any seat or verify run with `GridSize == 1`**, in the same shape
   `foliage.paint` already uses for its unprojected placement, naming the three fields that are fixed
   by construction. This is the whole ask if only one thing lands.
2. **Report where the column went.** The seat echo (`GroundPlacementHandler.cpp:1564-1574`) already
   carries `samples` and `undersideModel`; a `columnCentreCm` (or the `footprintHalfExtentCm` that
   `B-ground-instances-footprint-is-bounds-not-contact` asks for) makes the offset visible instead of
   requiring a `spatial.raycast` at the pivot to discover it.
3. **Omit rather than zero.** `groundSpreadCm: 0` and `undersideReliefCm: 0` at one column mean *not
   measured*, and read as *measured and flat*. This is the same "zero is absence" complaint
   `E-bounds-plane-fallback-has-no-batch-level-count` raises for the fallback rows; the fix shape
   should be shared.
4. **Correct the three doc lines** to say *world bounds centre*, not *centre* or *pivot-probe*, and
   add the sentence that the acceptance fields are vacuous at this setting.

## Not a duplicate of

- **`E-grounding-coverage-cannot-catch-a-single-point-rest`** (OPEN, Medium) — the closest sibling and
  deliberately not restated here. That ticket owns the *ratio* argument: `coverage` cannot express
  "enough columns" because its denominator can collapse, and its ask is an absolute
  `minSupportedColumns`. It arrives at `actorColumns: 1` from a mesh that only produced one column.
  **This ticket is the other route to the same degenerate state — the caller asking for it by
  parameter — and it carries two things that one does not:** the measured *location* of the column
  (152.7 cm p50 from the pivot, which is a wrong-place claim, not a wrong-count claim), and the fact
  that `groundSpreadCm` / `undersideReliefCm` collapse alongside `coverage`. A `minSupportedColumns`
  threshold would close that ticket and would **not** warn anyone here, because a caller who passed
  `samples: 1` deliberately will also set the threshold to 1.
- **`B-ground-instances-footprint-is-bounds-not-contact`** (OPEN, High) — filed alongside this one from
  the same pass. That is about the *extent* of a 3x3 or larger grid being the silhouette; this is
  about a 1x1 grid having no extent at all. They interact — `samples: 1` was adopted on this project
  precisely as an escape from that ticket's defect, and measures better on proposed dZ (+41.2 vs
  +196.0 p50) while producing the level's worst planted component — but a fixer could land either
  alone.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* — `B-ground-probe-hits-hull-not-render`,
`B-niagara-validate-green-while-component-inactive`, `B-mrq-render-result-omits-bitrate-and-size`:
the call succeeds, every number it reports is correct, and the output is wrong because the deciding
number was never reported. Here the deciding number is the XY the single column was taken at, and the
`samples` echo is the closest the response comes to it.

## Severity

**Medium.** Impact class is the rubric's Medium band twice over: *a readback omits a field and forces
a fallback* — the response never reports where the column landed, so discovering the 152.7 cm offset
requires a `spatial.raycast` at the pivot plus a source read of `GroundPlacementUtils.cpp:974-982` —
and *doable, but only via a documented workaround or a source dive*, since the workaround (do not use
`samples: 1`; if you must, verify externally) is only reachable once you know the fields are vacuous,
which no artefact states.

**Not High.** The rubric's High band is a silent false success or wrong data on a normal path. Every
number here is arithmetically correct for the sampling that was requested: `coverage` really is
1.0 over one column, `groundSpreadCm` really is 0 across one sample. The response asserts nothing
false; it omits the one field that would make the rest interpretable, and the docs use a word
("centre") that is technically true of the bounds and misleading about the object. That is omission
plus imprecision, not a lie — and rating it High would sort it above
`B-ground-instances-footprint-is-bounds-not-contact`, which is the defect that actually returns wrong
guidance on the default path.

**Not Low.** Low is *pure friction: docs, discoverability, naming, cosmetic*. The measured outcome is
208 badly planted instances that passed every published check, which is a wrong result rather than a
missing convenience — the distinction `E-python-get-editor-property-returns-live-view` and
`B-wiki-hism-recipe-resets-actor-transform` both drew when arguing a doc-shaped defect above the Low
band.

**Reach modifier declined.** `samples: 1` is a non-default opt-out on a verb that is not on the
every-session path, which by the rubric argues for a bump *down*. It is declined because the value is
reached for precisely when the default has already failed the caller — which on this project is every
trunked-tree scatter — so the population that hits it is not random.

## History
- `#1-one-column-is-vacuous-and-off-pivot` `OPEN` reporter — Filed from a planting-diagnosis pass on
  `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8), method in
  `Docs/map/tree-seating-on-slopes.md`, producers under `dev/planting/`. Mechanism re-derived at HEAD
  `6d0e91a3`: `samples` clamps to `MinGridSize = 1` (`GroundPlacementUtils.h:61`) at
  `GroundPlacementHandler.cpp:1419-1423` with no refusal, no warning and no response flag anywhere —
  `GridSize == 1` / `Samples == 1` / `== MinGridSize` have zero occurrences in the plugin source. At
  `Grid == 1` the grid loop (`GroundPlacementUtils.cpp:974-982`) reduces to `X = Origin.X, Y =
  Origin.Y` where `Origin = WorldBounds.GetCenter()` (`:922`), and `footprintInset` cancels out
  entirely. `ActorColumns` becomes 1 (`:998` sets `bHasActorGeometry` unconditionally in the
  `BoundsPlane` branch, tallied `:505-508`), so `coverage` (`:611-613`) is 1.0 for every row that
  reaches the gate (the 0 case exits earlier at `GROUND_NOT_FOUND`, `:725-731`), `groundSpreadCm`
  (`:580`) is identically 0 because `MinGround`/`MaxGround` are seeded from one column at `:549-550`
  and only widened at `:559-560`, `undersideReliefCm` (`:578`) collapses the same way, and
  `contactPoints` maxes at 1 against a `minContactPoints` that defaults to 1. Measured with
  `dev/planting/p_column.py` over 177 instances: the implied ground matches the AABB centre to
  **0.01 cm** and disagrees with the pivot by **rms 39.2 cm, max 172.7**; pivot-to-centre distance
  **p50 152.7 cm, max 324.6**, against a 100 uu contact radius. Field outcome: the one component
  seated at `samples: 1` (208 instances) is the level's worst and the only one with a positive median
  float (gap p50 +24.3, p90 +104.4, max +252.1), and it passed every published check.
  **Premise corrected during filing:** the report claimed `samples: 1` is recommended in the wiki. It
  is not — `docs/wiki-src/spatial.ground-placement.md:73` and `spatial.md:273` both describe it as an
  opt-out ("only if you want the old single-point behaviour back"), and
  `GroundPlacementHandler.cpp:784-785` names the boulder failure outright. The recommendation was the
  host project's own, now superseded. What survives: all three lines say "centre" or "pivot-probe"
  unqualified when the column is at the yaw-inflated world AABB centre, the `ground_instances`
  ParamSpec (`:1282`) drops even the boulder warning, and none of them says the acceptance fields
  become vacuous. Ask is a `warnings[]` entry at `GridSize == 1`, a `columnCentreCm` in the seat echo,
  omitting rather than zeroing the collapsed shape fields, and the three doc corrections — not a
  refusal, since the cheap mode is legitimate for a flat prop on flat ground. Deliberately NOT
  restating `E-grounding-coverage-cannot-catch-a-single-point-rest`'s ratio argument; that ticket's
  `minSupportedColumns` ask would not warn anyone here, because a caller who passes `samples: 1` sets
  the threshold to 1 too. Severity Medium on the readback-omits-a-field band; High declined because
  every reported number is arithmetically correct for the sampling requested; Low declined because the
  outcome is 208 wrongly planted instances, not missing convenience; reach bump declined downward with
  the argument stated.
