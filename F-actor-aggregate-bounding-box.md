---
id: F-actor-aggregate-bounding-box
title: "No scoped aggregate bounding box — get_bounding_box is single-actor and level.get_bounds is whole-level, so a group footprint must be unioned client-side"
status: OPEN
severity: Low
category: feature
tags: [actor-batch-set-asymmetry, get_bounding_box, level-get-bounds, bounds, footprint, batch]
encounters: 1
lastSeen: 2026-07-11T08:32:47.0194806+03:00
---

# No one-call footprint for a set / tag / folder

The common "how big is the group I just placed" intent has no one-call answer:

- `actor.get_bounding_box` (and `actor.get_transform`) accept only a single
  `actorName`.
- The only aggregate reader, `level.get_bounds`, is **whole-level** and cannot
  be scoped to a tag / folder / actor set — so on a populated map (e.g.
  `ExampleProjectWelcome`) it folds in every other actor, not just the kit.
  (It is also currently broken for a different reason — see
  `B-level-get-bounds-ignores-actors` — but even fixed it stays whole-level.)
- `spatial.measure_overlap` is a pairwise A/B test, not a set union.

So to report a group's footprint the caller must fetch N per-actor boxes and
union them by hand. This is **asymmetric with the set-aware methods already in
the `actor` namespace**: `actor.set_folder` takes `actorNames[]`, and
`actor.find_by_tag` / `actor.delete_by_tag` / `actor.spawn_batch` operate on the
whole set.

## What it should do

Accept a set selector — `actorNames[]`, a `tag`, or a `folder` — on
`actor.get_bounding_box` (or add a `folder`/`tag` filter to `level.get_bounds`)
and return one combined AABB (origin + extent) for the selected set. That turns
the "footprint of the arrangement" ask into a single call.

## Evidence (this task, outcome `clean` — batch-capability gap only)

Task: block out an arena and, once placed, "tell me the overall footprint of the
arrangement." The agent verified read-only that no scoped aggregate exists
(`level.get_bounds` is whole-map; `spatial.measure_overlap` is pairwise), then
issued **4 separate `actor.get_bounding_box` calls** (`Arena_Floor`
origin[0,0,0] extent[400,400,~0]; `Arena_Pillar_NE` extent[50,50,150];
`Arena_Pillar_SW` extent[50,50,150]; `Arena_Pedestal` extent[75,75,75]) and
unioned them by hand into the reported 800x800 cm footprint x 300 cm tall,
centered on origin. One scoped-aggregate call would have replaced the N fetches
plus the client-side union. The footprint came out correct — a convenience/
symmetry gap, not a bug.

severity rationale: impact=soft-blocker-with-workaround (N per-actor boxes unioned client-side) x reach=occasional block-out/measure path -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean arena block-out task (`ExampleProjectWelcome`). CallAnalyzer flagged 4 `actor.get_bounding_box` calls + a hand union to answer "overall footprint of the arrangement": `get_bounding_box` is single-actor, and the only aggregate (`level.get_bounds`) is whole-level and cannot be scoped to the 6-actor kit. Judge disposition: clean signal, no replay — footprint was computed correctly; purely a batch-convenience gap. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX found no scoped-aggregate-bounds ticket; `B-level-get-bounds-ignores-actors` is a distinct defect (whole-level get_bounds returning a zero box), not the missing set-scoping capability this ticket owns; `E-geometry-mesh-info-omits-bbox` is an asset-mesh readback, not an actor-set aggregate. Sibling `F-actor-batch-tag-set` shares the `actor-batch-set-asymmetry` family (single-actor verb where a set-aware one should exist) but is a distinct capability (tag write).
