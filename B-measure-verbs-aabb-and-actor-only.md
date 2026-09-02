---
id: B-measure-verbs-aabb-and-actor-only
title: "spatial.measure_distance / measure_overlap / verify_placement answer clearance from world AABBs over named actors only — wrong shape for a column or a dome, and structurally blind to every HISM/ISM instance"
status: OPEN
severity: Medium
category: bug
tags: [spatial, measure_distance, measure_overlap, verify_placement, aabb, clearance, hism, ism, instanced, proximity, false-clear]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# A clearance number that is neither the shape nor all of the geometry

Three `spatial` verbs answer "is anything in the way" from one primitive: the **world-space
axis-aligned bounding box of a named actor**.

    // World-space AABB of an actor, built the same way actor.get_bounding_box does:
    FBox MeasureActorBox(AActor* Actor)
    {
        Actor->GetActorBounds(false, Origin, Extent);
        return FBox(Origin - Extent, Origin + Extent);
    }
    -- Source/PinWright/Private/Handlers/Spatial/MeasureHandler.cpp:71-77

- `spatial.measure_distance` (`:104`) — `edgeGap = Sqrt(BoxA.ComputeSquaredDistanceToBox(BoxB))`
  (`:151`), `perAxisGap = |centerDelta| - (extentA + extentB)` (`:153`).
- `spatial.measure_overlap` (`:175`) — `BoxA.Intersect(BoxB)` (`:199`), penetration from the same
  extents (`:203-204`).
- `spatial.verify_placement`'s `noOverlapWith` check (`:342-370`) — the same box intersection per
  named actor.

Both operands come from `McpActorUtils::FindActorByName` (`:83`, and `:357` for
`noOverlapWith`), so the whole family is **actor-addressed**. The file's own header states the
contract and names the exception (`:9-13`): *"The first three read actor world-space AABBs the
same way actor.get_bounding_box does… find_clear_placement does not: an AABB per actor cannot
answer an occupancy SEARCH, so it queries the physics scene with a box overlap, which resolves
against ISM/HISM per-instance bodies as well as actors."*

## Two failure modes, and only one of them is documented

**1. An AABB is the wrong shape, and the docs warn about the safe direction only.**
`Saved/PinWright/wiki/spatial.measure_distance.md:7,29` publishes *"`edgeGap` (nearest
surface-to-surface gap, **0 when the AABBs overlap**)"* and *"AABBs bound a rotated non-box mesh
**looser** than its true silhouette, so expect a rotated mesh to read larger than it looks."* That
describes false **blocks**, which are conservative and survivable. The measured behaviour on this
map is worse in both directions — `Docs/map/atlantis-spec.md:347-349`, on a **dome**:

> An AABB is the wrong shape for a dome. `BLD_Dome_B1`'s box reports **0.0** clearance at
> f827 while the camera is 1044 uu outside the shell's plan silhouette; and it reports a
> comfortable figure at poses that are genuinely close. **Neither error is conservative.**

The same shape argument holds for a fluted Doric column: the box is the circumscribing square
prism, so the corners are empty and a pose beside a flute reads as contact. Reported today,
2026-09-02, while re-checking camera poses for a showcase render: a 0-uu hit against
`AVE_Col_N6` at a pose whose render shows the column clear. **That specific measurement is the
reporting agent's and I did not re-run it** — I verified only the mechanism above, and that the
spec already records the identical failure on `BLD_Dome_B1` with numbers.

**2. Instanced scatter is invisible, and nothing says so.** Neither the wiki page nor any error
mentions HISM/ISM. `GetActorBounds` over a scatter **holder** actor returns one box spanning
every instance it owns, so a holder is simultaneously (a) unable to report which instance is
near, and (b) a false block over its whole extent. `Docs/map/atlantis-spec.md:344-346`:

> It measured named actors only. The kelp, coral and rubble scatter are HISM instances on
> holder actors, and an actor-level OBB test sees one holder box spanning the whole
> 24000×24000 scatter — useless in both directions.

On this map that is most of the geometry: the same spec records 564 instances under six holder
actors (`:864-876`) and 51 HISM rock/coral instances in one search region (`:1037`). A clearance
question answered over named actors alone returns a clean number while ignoring the majority of
what is actually there.

## What does not close it

- **`spatial.find_clear_placement`** (shipped under `F-spatial-no-clear-footprint-search`) is
  instance-aware and physics-backed by construction (`MeasureHandler.cpp:11-13`), but it answers
  a different question: *find me a pose a footprint fits in*. It does not measure the clearance
  between a **given** pose or actor and its neighbours, which is what a camera-path gate, an
  acceptance check or a "did this move break anything" comparison needs.
- **`SystemLibrary.sphere_trace_multi` through `python.execute`** is the instrument that actually
  found the misses (`Docs/map/atlantis-spec.md:336-340`) and is what a clearance gate should be
  built on — but it is hand-rolled client-side code, not a verb, and its one-way segment form has
  its own direction-blindness on initial overlaps (`:351-362`). Cited here as the workaround, not
  as a fix.

**Fix direction (not prescribed):** make the shape and the scope of a clearance answer explicit
rather than implicit. Either (a) a shape-aware, instance-aware measurement verb backed by the
physics scene the way `find_clear_placement` already is — returning the blocking actor,
component **and instance index**; or (b) at minimum, make these three verbs say what they
measured: a `geometry: "actorAABB"` echo plus a warning when an operand owns instanced
components, so a caller learns from the response that the scatter was never considered.
Silence is what makes the number look like a measurement.

**Severity Medium**, argued: impact class is **High** — the instanced half is *silent wrong data
on a normal path*, with no warning in the response, no mention in the wiki, and scatter making up
most of the geometry it is asked about; a caller trusts a clean `edgeGap` and builds a camera
path on it. The AABB-shape half is milder because the wiki does publish the box contract, even
though it publishes only the conservative error direction. **Reach modifier −1**: these are rare
specialised verbs, not an every-session path. High × rare = **Medium**.

## Related

- `F-spatial-no-clear-footprint-search` (IN-REVIEW) — its history entry `#2` names this exact
  symptom, including the AABB-vs-dome and HISM-invisibility halves, but as *evidence*; its
  declared subject is the missing occupancy **search**, and its shipped fix
  (`spatial.find_clear_placement`) leaves `measure_distance` / `measure_overlap` unchanged. Not a
  dup — this ticket owns the existing verbs' geometry model.
- `F-spatial-swept-shape-query` (OPEN) — the path/volume half of the same absence.
- `B-ground-instances-footprint-is-bounds-not-contact` (IN-REVIEW) — the canonical "an AABB is
  not the contact geometry" argument on this board, scoped to the `ground_instances` seat grid.
- `B-hull-warning-blind-to-instanced-scatter` (IN-REVIEW) and `F-ism-per-instance-transforms`
  (IN-REVIEW) — the same instanced-scatter blind spot in the grounding and property families.
- `B-verify-grounding-maxgap-false-fail` (DONE) — same actor family (`AVE_Col_*`), same
  "measured the silhouette, not the thing" shape, on the vertical axis instead of the lateral one.

## History
- `#1-aabb-and-actor-only-clearance` `OPEN` reporter — `spatial.measure_distance` (`Handlers/Spatial/MeasureHandler.cpp:104`), `spatial.measure_overlap` (`:175`) and `spatial.verify_placement`'s `noOverlapWith` check (`:342-370`) all answer clearance from one primitive: `MeasureActorBox` = `Actor->GetActorBounds(false, Origin, Extent)` -> `FBox` (`:71-77`), with `edgeGap = Sqrt(BoxA.ComputeSquaredDistanceToBox(BoxB))` (`:151`), `perAxisGap` from the extents (`:153`) and `BoxA.Intersect(BoxB)` (`:199`). Both operands resolve through `McpActorUtils::FindActorByName` (`:83`, `:357`), so the family is actor-addressed; the file header states the contract and names `find_clear_placement` as the only instance-aware member (`:9-13`). Two failure modes. (1) Shape: the wiki publishes `edgeGap` "0 when the AABBs overlap" and warns AABBs bound a rotated mesh "looser than its true silhouette" (`Saved/PinWright/wiki/spatial.measure_distance.md:7,29`) — i.e. it warns only about conservative false BLOCKS, while `Docs/map/atlantis-spec.md:347-349` records both directions on `BLD_Dome_B1`: 0.0 clearance at f827 with the camera 1044 uu outside the shell's plan silhouette, and "a comfortable figure at poses that are genuinely close. Neither error is conservative." Reported 2026-09-02 on a fluted column: a 0-uu hit against `AVE_Col_N6` at a pose whose render shows it clear — that measurement is the reporting agent's and was NOT re-run here; only the mechanism was verified. (2) Instances: nothing in the wiki or in any error mentions HISM/ISM, and `GetActorBounds` over a scatter holder returns one box spanning every instance — `Docs/map/atlantis-spec.md:344-346` measured it as "one holder box spanning the whole 24000x24000 scatter — useless in both directions", over 564 instances under six holders (`:864-876`) and 51 HISM rock/coral instances in one search region (`:1037`). `spatial.find_clear_placement` does not close this: it is instance-aware but answers "find a pose a footprint fits in", not "how clear is this given pose". The workaround is hand-rolled `SystemLibrary.sphere_trace_multi` through `python.execute` (`Docs/map/atlantis-spec.md:336-340`), which is not a verb and whose one-way segment form is itself direction-blind on initial overlaps (`:351-362`). Ask: a shape-aware, instance-aware clearance measurement naming actor + component + instance index, or at minimum a `geometry` echo plus a warning when an operand owns instanced components, so the response says what was never considered.
