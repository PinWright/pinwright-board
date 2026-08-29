---
id: F-ground-instances-align-to-surface
title: "spatial.ground_instances declares no alignToSurface and no maxTilt while its sibling spatial.ground_actors declares both, so the plugin's only per-instance grounding verb computes the sampled ground normals and then discards them — every plant in an ISM/HISM scatter comes out perfectly vertical on every slope"
status: OPEN
severity: Medium
category: feature
tags: [spatial, ground_instances, ground_actors, ism, hism, instanced-static-mesh, scatter, vegetation, align-to-normal, maxTilt, parameter-parity, missing-parameter]
encounters: 1
lastSeen: 2026-08-29
---

# One verb tilts to the terrain, its per-instance twin cannot

`spatial.ground_actors` and `spatial.ground_instances` are the same solver over two footprint
sources — the handler's own comment says so
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Spatial/GroundPlacementHandler.cpp:1223-1226`:
*"The per-instance half of spatial.ground_actors ... Same solver, same required `surface`, same
'placed is a post-move measurement that passed' contract - only the footprint source and the write
differ"*). The parameter sets are not the same.

`spatial.ground_actors` (registration `:709`) declares surface-following:

- `alignToSurface` (`GroundPlacementHandler.cpp:811-814`) — *"Tilt the actor's +Z toward the AVERAGE
  of the sampled ground normals (more stable than any single normal), clamped by maxTilt."*
  Aliases `align_to_surface`, `alignToNormal`; default `false`.
- `maxTilt` (`:815-818`) — *"Ceiling in degrees on the alignToSurface tilt away from world up.
  Terrain normals on a cliff face approach horizontal; following one lays a prop on its side."*
  Aliases `max_tilt`, `maxTiltDegrees`; default `20`.

Both are read at `:905-909`, applied through `FGroundSeatConfig`, and echoed back at `:987-988`.

## The absence, shown by listing the whole parameter set rather than by grepping for two names

`spatial.ground_instances` registers at `GroundPlacementHandler.cpp:1235` with its `RPC_PARAMS` block
spanning `:1251-1329`. That block declares, in order and in full:

`surface` (`:1252`), `actorName` (`:1263`, via `ActorNameParamUtils::ActorNameParamReq`),
`component` (`:1265`), `indices` (`:1270`), `apply` (`:1273`), `samples` (`:1280`),
`footprintInset` (`:1286`), `seatPercentile` (`:1290`), `embedFraction` (`:1295`),
`embedDepth` (`:1299`), `minCoverage` (`:1302`), `minContactPoints` (`:1307`),
`contactTolerance` (`:1310`), `maxSeatError` (`:1314`), `limit` (`:1319`), `offset` (`:1322`),
`detail` (`:1324`).

Seventeen parameters. There is no rotation parameter of any kind — not `alignToSurface`, not
`maxTilt`, not an alias of either — anywhere in the block, and neither identifier appears anywhere
else in the `ground_instances` handler body (`:1330-1601`).

The generated wiki confirms the asymmetry from the caller's side, which is where it actually bites:
`Saved/PinWright/wiki/spatial.ground_actors.md:25-26` lists both parameters under **Parameters** and
again at `:51` in the overlay prose; `Saved/PinWright/wiki/spatial.ground_instances.md` lists the
seventeen above and neither of these two. A caller reading the two pages side by side sees a verb
pair that is documented as the same solve and is not.

## Measured consequence: the normal is computed, then dropped

This is not a knob that was never wired — it is a value the solve already has and throws away.
`GroundPlacement::SeatInstance`
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Spatial/GroundPlacementUtils.cpp:1403`) calls
`MeasureContactForBounds` at `:1469-1471` with an `OutColumns` array, and each `FGroundColumn`
carries the surface normal its trace returned. The seat then uses those columns for **height only**:
clearances are collected (`:1483-1491`), sorted, indexed by percentile (`:1502-1507`), and the
resulting move is a pure Z translation —

    FTransform Proposed = InstanceWorld;
    Proposed.AddToTranslation(FVector(0.0, 0.0, DeltaZ));

(`GroundPlacementUtils.cpp:1512-1513`). The rotation half of `InstanceWorld` is carried through
untouched on both the dry-run branch (`:1515-1524`) and the applying write
(`:1528-1537`, `UpdateInstanceTransform`).

So the documented ISM/HISM vegetation pipeline — the one `spatial.ground_actors`' own
`HOLDER_NOT_SEATABLE` refusal routes every scatter caller to (`GroundPlacementHandler.cpp:718-720`)
— produces a scatter that is correctly seated in Z and uniformly vertical in orientation, on
terrain of any slope. The averaged normal that would fix it existed in `SolveColumns` one statement
earlier.

**There is no recovery downstream, only reconstruction.** `actor.set_instance_transforms` can write
a rotation, but nothing hands the caller the normal to compute one from: `ground_instances` does not
report it, and `spatial.raycast` returns a *single* hit normal per ray, not the footprint average
this solver deliberately prefers (`GroundPlacementHandler.cpp:812-813`, *"more stable than any
single normal"*), and carries no `maxTilt` clamp. Reconstructing surface-following therefore costs
`samples^2` extra raycasts per instance plus a client-side re-implementation of the averaging and
the clamp — while the seat solve, which had the columns in hand, wrote a translation.

## Scope — `undersideModel` is deliberately NOT part of this ask

`ground_instances` also hardcodes the underside model rather than exposing `ground_actors`'
`undersideModel` parameter (`:791-796`). **That hardcode is correct and this ticket does not
question it.** It is echoed in the response, not hidden —

    SeatEcho->SetStringField(TEXT("undersideModel"), TEXT("bounds_plane"));

`GroundPlacementHandler.cpp:1573` — under a code comment at `:1570-1572` stating the reason: an
instance's underside cannot be probed per column, because
`UInstancedStaticMeshComponent::LineTraceComponent` answers from every instance body at once and
cannot attribute a hit to one of them. `SeatInstance` passes `EUndersideModel::BoundsPlane`
unconditionally (`GroundPlacementUtils.cpp:1470`). `F-ism-per-instance-transforms`' `#2` records the
same decision as a deliberate design addition its own signature had not anticipated. **A fixer must
not add `undersideModel` to this verb.** It is mentioned here only to show the parameter set was
read as a whole rather than grepped for two names.

**Residual, and it is a real one:** the reason is in the C++ comment and in the response echo, and
in neither of the two places a caller looks. `Saved/PinWright/wiki/spatial.ground_instances.md`
never mentions `undersideModel`, `bounds_plane`, or the `LineTraceComponent` attribution problem at
all — so a caller comparing the two pages sees a parameter the sibling has and this one lacks, with
no stated reason, and cannot tell a deliberate refusal from an omission. That belongs in the
`### spatial.ground_instances` overlay section of `Docs/wiki-src/spatial.md`, alongside whatever
this ticket lands.

**Fix:**

Declare `alignToSurface` + `maxTilt` on `spatial.ground_instances` with the same names, aliases and
defaults `ground_actors` uses, so the two verbs cannot drift in vocabulary the way they have drifted
in capability. The averaging and clamping code already exists on the actor path — the seam is
`SeatInstance`, which must compose the tilt onto `Proposed`'s rotation before the two returns at
`GroundPlacementUtils.cpp:1515-1524` and `:1528`, so the dry run predicts the same orientation the
applying path writes (the property the whole verb is built on).

Two instance-specific points a fixer needs and the actor path does not raise:

- The tilt must be composed onto the instance's **existing** rotation, not replace it — a scatter's
  per-instance yaw is the variation that makes it read as a scatter, and `set_instance_transforms`
  is not there to put it back.
- `maxSeatError`'s post-move readback (`:1314-1318`) re-measures clearance against the *seated*
  footprint. Tilting an instance changes its world bounds, so the re-measure must run against the
  tilted bounds or a correct surface-following seat will report `GROUND_SEAT_READBACK_MISMATCH`.

**Severity: Medium, argued.** Impact class is *not* the rubric's "High or Medium: hard blocker with
no workaround" — a workaround exists, it is just expensive and lossy: `spatial.raycast` per instance
for a single-ray normal, client-side averaging and clamping, then `actor.set_instance_transforms`.
That is the rubric's **Medium** verbatim — *"Doable, but only via a documented workaround, a source
dive, or many extra calls"* — and it is `samples^2` extra calls per instance across a scatter, which
is many. It is not High: nothing here is a silent false success or wrong data. `placed:true` from
this verb means exactly what it says (seated in Z, post-move measurement agreed); the verb never
claims to have aligned anything, and the response's `seat` echo (`:1565-1574`) lists no orientation
field to be wrong about. **Reach modifier declined in both directions, and named:** `spatial` has
eleven registered methods and `ground_instances` is not on the every-session path that would earn
the bump up — but it is also not a rare edge path, because it is the *only* per-instance grounding
verb and `HOLDER_NOT_SEATABLE` (`GroundPlacementHandler.cpp:718-720`) routes every ISM/HISM scatter
caller here by construction. Medium stands unmodified.

## Related

- `F-ism-per-instance-transforms` (IN-REVIEW, High) — shipped this verb along with
  `actor.get_instances` and `actor.set_instance_transforms`. Its `#2` records the `bounds_plane`
  decision this ticket declines to reopen; its "Proposed verb shape" section lists
  `{actorName, component?, indices?, surface, samples, seatPercentile, embed, apply:true}` and no
  orientation parameter, so `alignToSurface` was never proposed rather than proposed and dropped.
- `B-ism-undo-record-unsafe` (OPEN, High) — its `### 2` heading is *"`spatial.ground_instances`
  never echoes `space`"*, and the body notes the verb *"declares no `space` parameter at all"*. Same
  theme from a different angle: the per-instance verb shipped with a narrower parameter surface than
  its siblings, and each missing one is being found separately. A fixer touching this verb's
  `RPC_PARAMS` block should read that ticket before deciding what else belongs there.
- `E-ground-preset-excludes-only-foliage-actors` — the `surface` side of the same verb pair.
- `F-scatter-layout-verb` (DONE) — `spatial.scatter_layout` produces the XY lattice this verb then
  seats; neither end of that pipeline can tilt an instance to the ground it lands on.

## History
- `#1-align-and-tilt-absent-on-instances` `OPEN` reporter — Confirmed by listing
  `spatial.ground_instances`' complete declared parameter set (`GroundPlacementHandler.cpp:1251-1329`,
  seventeen params, enumerated above) rather than by grepping for two names: no `alignToSurface`, no
  `maxTilt`, no alias of either, and neither identifier anywhere in the handler body `:1330-1601`.
  `spatial.ground_actors` declares both at `:811-818`, reads them at `:905-909` and echoes them at
  `:987-988`. Caller-side asymmetry confirmed against the generated wiki
  (`spatial.ground_actors.md:25-26` and `:51` vs `spatial.ground_instances.md`, which lists
  neither). Traced the discard: `SeatInstance` (`GroundPlacementUtils.cpp:1403`) collects
  `SolveColumns` with their normals at `:1469-1471`, uses them for clearance only (`:1483-1507`),
  and emits a pure Z translation at `:1512-1513`, carrying the instance's original rotation through
  both the dry-run return (`:1515-1524`) and the `UpdateInstanceTransform` write (`:1528-1537`).
  Also checked and NOT asking for: the `undersideModel` hardcode at `:1573` is echoed in the
  response and justified in a comment at `:1570-1572` (per-instance underside is unprobeable because
  `LineTraceComponent` answers from every body at once), matching `F-ism-per-instance-transforms`'
  `#2` — recorded here as a wiki documentation residual only, since the generated
  `spatial.ground_instances.md` states neither the hardcode nor the reason.
