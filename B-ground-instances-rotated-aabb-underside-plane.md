---
id: B-ground-instances-rotated-aabb-underside-plane
title: "The underside plane `spatial.ground_instances` seats to is the ROTATION-INFLATED world-AABB minimum, which for a tumbled prop sits well below the mesh's real lowest point — so `seatPercentile` has no fixed zero across instances and the value that reads as a gentle sink LIFTS an already-bedded rock: measured 245 of 256 lifts at 0.4 and 247 sinks at 0.9 on the same component, every row `pass: true`, `coverage: 1.00`, `undersideReliefCm: 0` — and the response publishes no absolute Z from which a caller could see it coming"
status: OPEN
severity: Medium
category: bug
tags: [spatial, ground_instances, ism, hism, instanced-static-mesh, scatter, vegetation, rocks, debris, bounds, aabb, rotation-inflated, underside, bounds-plane, seat-percentile, readback, zero-is-absence, placement, level-building]
encounters: 1
lastSeen: 2026-08-29T20:50:00+03:00
---

# The box bottom is not the mesh bottom, and it drops further the more the instance is tumbled

`spatial.ground_instances` models an instance's underside as a flat plane at the **world-space
axis-aligned bounding box minimum**. `FBox::TransformBy` re-fits that box around the *rotated* mesh,
so the plane sinks below the mesh's real lowest point by an amount that grows with pitch and roll —
and every quantity the response publishes is a *difference* against that plane, never the plane
itself.

Two consequences, and the second is the one that cost real work:

- **`seatPercentile` has no fixed zero.** Its scale is anchored to a plane whose offset from the
  mesh differs per instance, so the same value means a different depth on every rock in the same
  scatter.
- **A low percentile lifts an already-bedded prop**, which is the opposite of what the parameter's
  own documentation leads a caller to expect (`seatPercentile: 0` is described as the first-contact
  "balanced boulder" rest, i.e. the *highest* seat). Measured: `0.4` proposed a **lift** on 245 of
  256 rows; `0.9` proposed a **sink** on 247 of the same 256.

## Scope, stated first

**This ticket is about the four mesh classes `B-ground-instances-footprint-is-bounds-not-contact`
explicitly excludes.** That ticket measured bounds/contact **area** ratios and concluded the AABB is
"a good model for four of these six meshes (0.98-2.49x)" — rocks, slabs, logs, boulders — and scoped
itself to the 46.7-80.8x trunked-tree class. That conclusion is correct about the **XY** footprint
and wrong about the **Z** term of the same box. The four low-ratio classes still seat wrongly, in the
opposite direction, whenever the instance carries pitch or roll.

A `contactRadius` / `footprintSource` fix as asked for there changes which XY the columns are probed
at. It does not move `BottomZ`. Neither ticket subsumes the other.

## Mechanism, re-derived at PinWright HEAD `0f4f9594`

All paths in `Plugins/PinWright/Source/PinWright/Private/Handlers/Spatial/`.

**The plane is the rotated box's floor.** `SeatInstance` builds the world AABB and the sampler
derives everything from it:

    GroundPlacementUtils.cpp:1450   const FBox InstanceBounds = Mesh->GetBounds().GetBox().TransformBy(InstanceWorld);
    GroundPlacementUtils.cpp:922-923   const FVector Origin = WorldBounds.GetCenter();
                                       const FVector Extent = WorldBounds.GetExtent();
    GroundPlacementUtils.cpp:937       const double BottomZ = Origin.Z - Extent.Z;

`FBox::TransformBy` returns the axis-aligned hull of the rotated box, so `Extent.Z` is not
`meshHalfHeight * scaleZ` — it is `|M[0][2]|*ex + |M[1][2]|*ey + |M[2][2]|*ez` over the mesh's local
half-extents, which is `>= ez` for every rotation and strictly greater for every rotation that is not
axis-aligned. **Pure yaw does not do this** (`M[0][2] = M[1][2] = 0`, `M[2][2] = 1`): the inflation is
entirely a pitch/roll effect.

**Every column then gets that one number as its underside.** The instance path has no other model —
`BoundsPlane` is a literal at both measurement sites (`GroundPlacementUtils.cpp:1471`, `:1564`) and
the handler assigns it unconditionally with no parameter read
(`GroundPlacementHandler.cpp:1487`, rationale `:1475-1477`):

    GroundPlacementUtils.cpp:996-1000   else
                                        {
                                            Column.bHasActorGeometry = true;
                                            Column.UndersideZ = BottomZ;
                                        }

**The seat then solves to that plane.** `GroundPlacementUtils.cpp:1510`,
`const double DeltaZ = -(SeatClearance + EmbedCm);`, where `SeatClearance` (`:1502-1508`) is the
`seatPercentile`-th order statistic of `UndersideZ - GroundZ` over the columns. With `UndersideZ` constant, that is a
percentile of the **ground alone**, offset by a plane whose distance below the mesh nobody can see.

**And the plane's Z never leaves the handler.** `FGroundColumn` is a local in
`MeasureContactForBounds` (`GroundPlacementUtils.cpp:971`), returned only through the optional
`OutColumns` (`:1469`, `:1561`) which `SeatInstance` consumes and discards. `Column.UndersideZ` and
`BottomZ` appear in **zero** `SetNumberField` calls anywhere in the repo. What the response does
publish are differences only (`AggregateColumns`, `GroundPlacementUtils.cpp:577-580`):

    MaxGapCm          = MinUnderside - MinGround
    MinGapCm          = min over columns of (UndersideZ - GroundZ)
    UndersideReliefCm = MaxUnderside - MinUnderside      // identically 0 under BoundsPlane
    GroundSpreadCm    = MaxGround   - MinGround

serialized per row inside `contact` at `GroundPlacementHandler.cpp:511`, `:517`, `:518`. **No absolute
ground Z is emitted either**, so the pair is under-determined: the caller gets `UndersideZ - GroundZ`
and neither term. The batch `seat` echo names the model as a constant string
(`GroundPlacementHandler.cpp:1631`) and no Z. Reconstructing the plane offline needs the mesh's local
bounds, which this response does not carry — it publishes `component`, `componentClass` and
`instanceCount` and nothing about the mesh (`InstancedMeshUtils.h:257-260`).

## Measured

`/Game/Maps/PW_VegetationTest`, host `EAContentExamples58`, UE 5.8. `InstancedFoliageActor_0` /
`FoliageInstancedStaticMeshComponent_43`, **324 `SM_Rock` instances**, `surface: {preset: "landscape"}`,
`apply: false`. Rows are capped at 256 by `GroundRpcMaxDetailRows` (`GroundPlacementHandler.cpp:64`),
so every count below is out of the **256 reported rows**, not out of 324. Payloads spilled to
`Saved/PinWright/HttpResponses/20260829T162018Z/`; statistics re-derived from them, not relayed.

| parameters | proposed dZ p50 | min | max | lifts | sinks |
|---|---:|---:|---:|---:|---:|
| `seatPercentile 0.4, samples 5, embedDepth 5, embedFraction 0` | **+3.9** | -1.7 | +13.6 | **245** | 11 |
| `seatPercentile 0.9, samples 5, embedDepth 4, embedFraction 0` | **-11.4** | -45.5 | +3.0 | 9 | **247** |

The same reversal on the two sibling meshes at `0.9`: `SM_Rock1` (`...Component_42`, 284 instances)
253 sinks / 3 lifts, p50 -10.0; `SM_Rock2` (`...Component_40`, 268) 251 sinks / 5 lifts, p50 -10.2.
**`pass: true` on every row of every one of those runs**, `coverage` minimum 1.000,
`contactPoints` p50 19 of 25 sampled columns.

**The rows describe a flat-bottomed object, and the object is a tumbled rock.** On all 256 rows of
the `0.4` run, `undersideReliefCm` is exactly `0` and `maxColumnClearanceCm` is bit-identical to
`maxGapCm` — the shape term is not merely unmeasured, it is asserted absent. The instances it is
asserted about carry `|pitch|` p50 16.1 deg / max 53.2 and `|roll|` p50 16.0 / max 51.0; the angle
between the instance's own +Z and world up is p50 **26.6 deg**, p90 44.7, max 58.6. **157 of the 256
carry more than 20 degrees on at least one axis and not one of them is upright.**

**How far below the mesh the plane sits, computed.** `SM_Rock`'s local vertex AABB from its own LOD0
vertices (`dev/planting/p_meshprofile.py`, output `dev/planting/out/meshprofile.json`) is
`zMin -18.30`, `zMax 25.00`, `boundsExtentXY (21.30, 21.30)` — half-extents `(21.30, 21.30, 21.65)`,
i.e. very nearly a cube, which is the *least* favourable case for this mechanism. Applying each row's
own rotation and scale (all 256 are uniformly scaled, 0.86-2.82), the rotation term alone lowers the
modelled plane by **p50 16.7 cm, p90 27.7, max 40.2** relative to where the same instance unrotated
would put it — against a mesh whose entire unscaled height is 43.3 uu. Spearman correlation between
that computed drop and the verb's own reported `penetrationCm` is **+0.495** over the 256 rows
(positive as the mechanism predicts, and only moderate because terrain slope is the other term).

**The verb's own numbers agree.** On the `0.4` run `penetrationCm` is p50 **25.1 cm**, p90 45.5,
max 70.7 — the modelled bottom face is a quarter of a metre *underground* at the deepest column,
while the rocks it describes were sitting on or above that terrain (76.2% of this class measured
daylight under them before the pass). The verb is not confused about the box; the box is a quarter
of a metre below the rock.

**Field consequence.** Seating the three rock components at `0.9` moved 905 of 1018 instances and took
measured float from 76.2% -> 55.2% (`SM_Rock`), 100% -> 94.0% (`SM_Rock1`) and 100% -> 81.3%
(`SM_Rock2`), gap p50 +12.6 -> +1.5, +21.5 -> +9.7 and +17.7 -> +5.9 cm respectively
(`dev/planting/p_gap.py`, same metric both sides, `out/gap_before.json` vs `out/gap_after.json`).
That is the *good* end of the parameter, reached only after the reversal was diagnosed. The first
apply, at percentiles chosen from the documented reading of the parameter, left **105 instances
measured worse than before** and 38 had to be handed back from a pre-recorded original set.

One component is the clean control: `...Component_37` (48 `SM_Rock`) is 0% floating and bedded at
p50 -23.9, and at `seatPercentile 0.9` the dry run proposes a **lift on 48 of 48**, +22.2 to +75.0 cm.
Correctly-planted props are exactly the ones this lifts.

## Ask

**Publish the plane.** A per-instance `undersideZCm` (and, for the same cost, `groundZCm` — the
`MinGround` the seat solved against) inside the existing `contact` object, beside `maxGapCm` at
`GroundPlacementHandler.cpp:511`. Both are already in hand at that point: `MinUnderside` and
`MinGround` are locals of `AggregateColumns` (`GroundPlacementUtils.cpp:577-580`) that are currently
subtracted and thrown away. With either one absolute number the caller can compute the mesh-vs-box
discrepancy from the mesh bounds and predict the direction of the move before running anything.

Cheaper and weaker, if the absolute Z is unwanted in the payload: publish
`boundsRotationInflationCm` — `Extent.Z - meshHalfHeight * scaleZ`, one subtraction — which is the
whole of the surprise in one number and needs no absolute coordinate.

Stronger and separable: make the plane something other than the rotated hull's floor, e.g. the
lowest transformed **vertex** of the mesh's lowest LOD, computed once per (mesh, rotation) and cached.
That would give `seatPercentile` a fixed zero. It is a larger change than this ticket asks for and is
not required to close it.

**Not asked for here: a per-column probeable underside.** `GroundPlacementHandler.cpp:1475-1477`
states why an instance cannot have one (`UInstancedStaticMeshComponent::LineTraceComponent` answers
from every body at once), `F-ground-instances-align-to-surface` examined and accepted that, and so
does this ticket. The complaint is not that the model is a flat plane; it is that the plane's height
is unknowable from the response.

## Not a duplicate of

- **`B-ground-instances-footprint-is-bounds-not-contact`** (OPEN, High) — the XY extent the columns
  are sampled over. Disjoint axis, disjoint mesh classes, disjoint fix. See § *Scope*; that ticket's
  `#2` carries the cross-reference from its side.
- **`E-ground-instances-embed-fraction-of-bounds-height`** (OPEN, Low) — **the closest sibling, and
  the same root cause on a different term.** That ticket is `embedFraction`'s *height* multiplier
  being the rotation-inflated `2 * Extent.Z`; this is the *floor* `Origin.Z - Extent.Z` of the same
  re-fitted box. Both make a parameter mean different things on two instances of the same mesh. They
  are filed apart because the remedies do not overlap: that one is a documentation-and-defaults
  question about a knob whose value does reach the caller, this one is a quantity that reaches nobody.
  A fixer working either should read both.
- **`E-bounds-plane-fallback-has-no-batch-level-count`** (OPEN, Medium) — the mesh-profile probe
  *falling back* to a bounds plane on `verify_grounding` / `ground_actors` without a batch count. On
  `ground_instances` there is no fallback; `bounds_plane` is the only model. The zeroed
  `undersideReliefCm` reported above is the same symptom down a different path, already noted as such
  by the footprint ticket.
- **`B-ground-instances-single-sample-degenerate`** (OPEN, Medium) — `samples: 1` making the quality
  fields vacuous. Every run measured here used `samples: 5` with 25 sampled columns and
  `contactPoints` p50 19, so none of these numbers is degenerate; they are fully-supported
  measurements of the wrong solid.
- **`F-ground-instances-move-clamps`** (OPEN, Medium) — filed from the same pass. That ticket asks for
  a way to **bound** the move; this one asks for a way to **understand** it. A `maxLift` clamp would
  have prevented the damage measured here without disclosing anything, and disclosure would have let
  the caller build the clamp themselves — so they mitigate each other and neither is the other's fix.

## Severity

**Medium**, on the rubric's soft-blocker band, both clauses of which apply: *"doable, but only via a
documented workaround, a source dive, or many extra calls"* — diagnosing why `0.4` lifted required
reading `FBox::TransformBy` and the `BoundsPlane` branch, and recovering cost a full extra
measurement pass plus a revert from pre-recorded originals — and *"a readback omits a field and forces
a fallback"*, which is the ask, precisely.

**Not High, and the argument matters because the sibling ticket is High.** The High band is silent
wrong data the caller builds on. The response here is not wrong: it names `undersideModel:
"bounds_plane"` on every row, every published difference is correct about that plane, and a dry run
reports `proposedDeltaZCm` truthfully, so the lift **is** detectable before it is applied — 245/256
is a number this project read off the verb's own output. What is missing is *why*, and the ability to
predict it without spending the dry run. The magnitude separates it too: the footprint ticket's error
is p50 +196 cm on a 19 m tree, this one is p50 +3.9 cm on a 1 m rock. Same mechanism family, two
orders of magnitude apart in consequence.

**Not Low.** `undersideReliefCm: 0` and `maxColumnClearanceCm == maxGapCm` are not cosmetic: they are
positive assertions of a flat underside on 256 rows describing objects tilted a median of 26.6
degrees, and they are what a reviewer checking "did this seat cleanly" reads.

**Reach modifier declined, in the direction that would raise it.** The case for a bump is real:
`spatial.ground_instances` is the plugin's only per-instance grounding verb and `HOLDER_NOT_SEATABLE`
(`GroundPlacementHandler.cpp:718` on `ground_actors`, `:1038` on `verify_grounding`) routes every
ISM/HISM caller here by construction. It is declined because the affected population is precisely the
instances carrying **pitch or roll** — pure yaw leaves `Extent.Z` untouched, as shown above — and an
upright scatter, which is most vegetation traffic on this verb, is unaffected. That makes this a
defect on the tumbled-debris subset rather than on the verb's normal path, and Medium is the right
band without the bump.

## History
- `#1-plane-is-the-rotated-box-floor` `OPEN` reporter — Filed from a bulk instance re-seat on
  `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8); method and full tables in
  `Docs/map/tree-seating-on-slopes.md` § *Bulk re-seat*, producers under `dev/planting/`. Mechanism
  re-derived at PinWright HEAD `0f4f9594` rather than relayed: the plane is `BottomZ`
  (`GroundPlacementUtils.cpp:937`) off `Mesh->GetBounds().GetBox().TransformBy(InstanceWorld)`
  (`:1450`), written into every column's `UndersideZ` in the `BoundsPlane` branch (`:996-1000`), with
  `BoundsPlane` a literal at `:1471`/`:1564` and assigned unconditionally at
  `GroundPlacementHandler.cpp:1487`; the seat is `DeltaZ = -(SeatClearance + EmbedCm)`
  (`GroundPlacementUtils.cpp:1510`); and `Column.UndersideZ` / `BottomZ` reach **zero**
  `SetNumberField` calls anywhere in the repo, so only differences are published
  (`AggregateColumns`, `:577-580`, serialized `GroundPlacementHandler.cpp:511`, `:517`, `:518`) with
  no absolute ground Z either — the pair is under-determined and the mesh bounds are not in the
  response (`InstancedMeshUtils.h:257-260`). **Two premises from the report this was filed from were
  corrected before filing.** (1) "Measured on 466 rotated rocks" — the dry runs are per component and
  capped at 256 rows (`GroundPlacementHandler.cpp:64`), so 245/256 and 247/256 are out of the
  **reported rows** of `...Component_43`'s 324 instances, not out of the class's 466. (2) "a caller
  cannot predict that a low seatPercentile will LIFT an already-bedded prop" — a caller **can**,
  from `proposedDeltaZCm` on a dry run (`GroundPlacementHandler.cpp:1593`), which is how this
  project measured it; what cannot be predicted is the direction without spending that dry run, and
  what cannot be recovered at all is the plane. Severity was set from the corrected reading, and the
  correction is why this is Medium rather than High. Measured evidence re-derived from the spilled
  payloads in `Saved/PinWright/HttpResponses/20260829T162018Z/`: `0.4` lifts 245/256 (p50 +3.9, max
  +13.6), `0.9` sinks 247/256 (p50 -11.4, min -45.5), `pass: true` and `coverage: 1.000` on every
  row of both, `undersideReliefCm` exactly 0 and `maxColumnClearanceCm` bit-identical to `maxGapCm`
  on all 256, `penetrationCm` p50 25.1 / max 70.7, tilt of instance +Z from world up p50 26.6 deg
  with 157/256 beyond 20 deg on an axis and none upright. Rotation-induced drop of the plane computed
  from `SM_Rock`'s own local half-extents `(21.30, 21.30, 21.65)`: p50 16.7 cm, max 40.2, Spearman
  +0.495 against the verb's reported `penetrationCm`. Explicitly NOT reopening the `bounds_plane`
  hardcode (`GroundPlacementHandler.cpp:1487`, rationale `:1475-1477`), which
  `F-ground-instances-align-to-surface`
  already accepted. Scope correction to `B-ground-instances-footprint-is-bounds-not-contact` recorded
  on both tickets: its "good model for four of these six meshes" holds in XY and fails in Z, and this
  ticket is exactly those four classes.
