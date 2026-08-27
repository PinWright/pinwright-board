---
id: F-spatial-no-clear-footprint-search
title: "No occupancy query — 'does a WxH footprint fit near here, and if not where does it' has to be hand-rolled as OBB math against a scraped actor list"
status: OPEN
severity: Medium
category: feature
tags: [spatial, placement, occupancy, clearance, footprint, level-building, verify_placement, workaround]
encounters: 2
lastSeen: 2026-08-27T23:55:00.0000000+05:00
---

# `spatial` can verify a placement you already chose, but cannot find one

Every `spatial` verb that knows about occupancy needs the answer as input:

- `spatial.verify_placement` `{noOverlapWith:[names]}` — you must already have a
  transform **and** name the actors it must miss.
- `spatial.measure_overlap` / `spatial.measure_distance` — strictly pairwise A/B.
- `spatial.place_relative` / `place_on_surface` / `ground_actors` — solve the
  **Z/anchor** of a spot you have already picked in XY; none of them refuses a
  spot because something else is standing there.
- `spatial.raycast` — one ray, and a ray answers "what is at this point", not
  "is there a rectangle of clear ground".

So the first question in any placement task — *is there room here for this
asset, and if not, where is the nearest place there is?* — has no verb. The
caller has to leave the plugin: scrape every candidate obstacle out of the level,
rebuild each one's footprint client-side, and run their own OBB/capsule/circle
clearance search.

## What it should do

A read-only `spatial.find_clear_placement` (or an occupancy mode on
`verify_placement`), taking:

- the footprint — `{width, depth}` or, better, `assetPath` / `actorName` so the
  verb reads the mesh's own bounds instead of the caller re-deriving them;
- a search region (`center` + `radius`, or a min/max box) and a `yawStep`;
- exclusion inputs the caller already thinks in: `ignoreActors`, `onlyClasses`,
  a minimum `clearance`, and optionally a keep-out polyline (a camera path is
  the recurring one).

Returning the ranked clear poses with the measured clearance and the **name of
the actor that binds** each rejected one. The binding-actor name is the load-
bearing half: "0 poses fit" is not actionable, "0 poses fit, the binder is
always the HISM scatter" is.

Note the verb must see **HISM/ISM instances**, not just actors. Instanced
scatter is what actually fills a dressed level, and it is invisible to every
name-based occupancy input the namespace currently offers — `noOverlapWith`
takes actor names, and a HISM holder's single AABB spans the whole scatter, so
naming it is useless (see `B-ground-actors-prefix-captures-foreign-actors` for
the sibling trap the same holder shape already caused).

## Evidence (this task — outcome: placed nothing, correctly)

Task: place `/Game/Atlantis/Meshes/SM_Stair_Block` (4004 × 2272 × 1023) as
collapsed masonry inside an existing temple-debris field in `/Game/Maps/Atlantis`,
or report that it does not belong there. Answering that is exactly an occupancy
question, and it cost:

- **2 `python.execute` scrapes** to get what the verbs will not give — one for the
  `Rubble_*` transforms, one walking `InstancedStaticMeshComponent` instance
  transforms because there is no other way to see scatter occupancy;
- **~150 lines of hand-rolled 2-D geometry** offline (OBB corners, SAT overlap,
  segment-segment distance, point-in-polygon, capsule and circle cases) plus a
  17 328-pose grid search;
- **two real bugs in that hand-rolled math**, both of which produced confident
  wrong answers before they were caught: an unclamped projection parameter in
  segment-segment distance that measured to the infinite *line* (every camera-path
  clearance read 0), and an `overlap == 0.0` result passed by a `< 0` reject
  filter (poses sitting **on** the temple podium were reported as the best
  candidates). Both were found only by unit-testing the helpers against known
  answers — i.e. the workaround silently produces plausible wrong geometry, which
  is the argument for the verb existing.

The conclusion the search produced was sound and is now recorded in the host
project's map spec: the footprint fits nowhere in the quadrant (0 clear poses of
17 328 at the as-built size; the nearest fully clear pose is 4943 uu away, past
a dome ruin, in open sand), so nothing was placed. But the plugin contributed
none of that reasoning — it supplied only the raw transforms.

severity rationale: impact=no verb at all for a first-order level-building
question, workaround is client-side computational geometry that demonstrably
yields wrong answers when hand-rolled × reach=every dress-a-populated-level
placement task -> Medium

## Cross-ref (adjacent, kept separate)

- `F-spatial-raycast-no-batch-multi-origin` (OPEN) — asks for batched rays / a
  `find_flat_region` surface scan. That is about the **ground**: flatness and
  extent of terrain. This ticket is about **occupancy**: what is already standing
  on that ground. In the evidence task the terrain was never the blocker; the
  other actors were. A fix to either leaves the other question unanswered.
- `F-actor-aggregate-bounding-box` (OPEN) — one call for the union footprint of a
  set. Same "footprint" vocabulary, but it measures a group you already have;
  this verb searches for a place to put one more.
- `B-ground-actors-prefix-captures-foreign-actors` (OPEN) — the HISM-holder
  bounding-box trap this verb would have to avoid on the read side.

## History
- `#1-initial-report` `OPEN` reporter — Filed from a level-dressing task in `EAContentExamples58` (`/Game/Maps/Atlantis`): decide whether a 4004 × 2272 × 1023 collapsed-stair mesh could be composed into an existing temple-debris field. The question is pure occupancy and no `spatial` verb answers it — `verify_placement` needs the transform and the blocking actor names up front, `measure_overlap`/`measure_distance` are pairwise, `place_relative`/`place_on_surface`/`ground_actors` solve Z for a spot already chosen, `raycast` is one ray. Worked around with 2 `python.execute` scrapes (including a walk of `InstancedStaticMeshComponent` instance transforms, the only way to see scatter occupancy) plus ~150 lines of client-side OBB/SAT/capsule geometry over a 17 328-pose grid. Two bugs in that hand-rolled math each produced confident wrong output first (unclamped segment-segment projection measuring to the infinite line — every camera-path clearance read 0; and an `overlap == 0.0` result passing a `< 0` reject filter — poses on top of the temple podium ranked best), caught only by unit-testing the helpers, which is the case for the capability living in the plugin. Dedup: ripgrep across the board for clearance / free-space / OBB / candidate-placement found no occupancy-search ticket; `F-spatial-raycast-no-batch-multi-origin` is terrain flatness not actor occupancy, `F-actor-aggregate-bounding-box` measures an existing set. Proposes a read-only `spatial.find_clear_placement` taking a footprint (or `assetPath`), a search region + `yawStep`, `ignoreActors`/`onlyClasses`/min-clearance and an optional keep-out polyline, returning ranked clear poses with measured clearance and the **binding actor** for rejected ones, and required to see HISM/ISM instances rather than only actors.

- `#2-camera-corridor-clearance` `OPEN` reporter — Second encounter from the same map, the other
  half of the same missing capability: gating a **camera path** (the keep-out-polyline case this
  ticket already names) rather than placing an asset. Re-keying the orbit third of
  `/Game/Atlantis/Cine/LS_Atlantis_Flythrough` needed "is any geometry within 900 uu of this
  polyline, and which actor binds it" at 1201 sample points. Nothing in `spatial` answers it:
  `measure_distance` is actor-AABB pairwise and an AABB is the wrong shape for the dome that
  actually bounds this path; `verify_placement` needs the blocking actor names up front, which is
  the answer; and the scatter that binds most of the path is HISM instances, invisible to every
  actor-level verb. Hand-rolled again in `python.execute` over
  `SystemLibrary.sphere_trace_multi`.
  **The hand-rolled instrument produced a false "clear", and so did the project doc's recommended
  form of it.** A one-way segment sweep is direction-blind on initial overlaps: `A -> B` at
  radius 900 returned `[]` while `B -> A` on the same segment returned
  `[STA_FallenB_03, 855 uu]`. A zero-length vertical probe (`p` to `p + 1 uu` in Z) shares the
  blindness — 931.8 uu reported where the real surface point is 855.1. On the strength of the
  one-way sweep `Docs/map/atlantis-spec.md` had published "the retuned path measures 0 segments
  under 900 uu"; a 6-direction probe measuring true `|p − impact_point|` finds **167 frames under
  900**, tightest 573 uu. That is the same failure mode as encounter #1 (hand-rolled clearance math
  is confidently wrong before it is right), reached from a different direction, and it is the
  argument for the verb: the correct query is not expressible, so every caller re-derives it and
  some of them ship the wrong answer as a green metric.
  Two incidental UE 5.8 notes for whoever implements it: `FHitResult` has no
  `get_editor_property('hit_actor')` (raises; `hit.to_dict()['hit_actor']` works), and
  `sphere_trace_multi` returns `Array[HitResult] or None` — `None`, not an empty array, on a clean
  sweep. Ask for `minClearance` + the binding actor name on a **polyline** input, not only a
  footprint search.
