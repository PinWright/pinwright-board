---
id: F-actor-list-omits-location
title: "actor.list can't project location/transform — bulk spatial reconnaissance (find a cluster center) costs one actor.get per sampled actor"
status: OPEN
severity: Low
category: feature
tags: [actor-batch-set-asymmetry, actor-list, actor-get, projection, location, transform, bulk, spatial]
encounters: 2
lastSeen: 2026-09-06T00:00:00Z
---

# No bulk actor-location read — spatial reconnaissance is one `actor.get` per actor

Reasoning about where actors sit relative to each other — "find the dense cluster
of props and pick an anchor near its center", "which actor is nearest X", "what's
in this region" — has **no one-call route**. Every actor reader that returns a
world location is single-actor, and the only bulk enumerate verb can't project
position:

- `actor.get` / `actor.get_transform` / `actor.get_bounding_box` accept only a
  single `actorName`. To sample the layout you call one per actor.
- `actor.list` is the bulk enumerate verb and now has a `fields` projection
  (shipped by `E-actor-list-no-limit-spills #4`), but its allow-list is
  **`label` / `name` / `path` / `class` only** — no `location` / `transform` /
  `bounds`. So it tells you *what* actors exist, never *where* they are.
- `system.inspect.list_objects` / `level.get_actors` return identities, not
  transforms.

So to triangulate a spatial grouping the caller must fan out N single
`actor.get` reads and cluster the coordinates client-side. This is
**asymmetric with the set-aware verbs already in the `actor` namespace**
(`actor.set_folder` takes `actorNames[]`; `actor.find_by_tag` /
`actor.delete_by_tag` / `actor.spawn_batch` operate on the whole set) — only the
*spatial read* is single-actor-or-nothing.

## What it should do

Either (cheapest, preferred):

- **Add `location` (and optionally `transform` / `bounds`) to `actor.list`'s
  `fields` projection allow-list.** The projection mechanism already exists; this
  extends the allow-list past the four identity fields so `actor.list
  {fields:["label","location"]}` returns every actor's position in one call —
  turning "where is everything" into the same single enumerate call that already
  answers "what is everything". (This is the direct actor-namespace analogue of
  the material ask in `E-material-list-all-omits-position` — a list-all surface
  that carries every other field but drops the coordinates the intent needs.)

or:

- a batch transform getter keyed by a set selector (`actorNames[]` / `class` /
  `tag` / `folder`) returning `[{name, location, rotation, scale}]` for the set.

## Distinct from

- `F-actor-aggregate-bounding-box` (OPEN, same `actor-batch-set-asymmetry`
  family) — that returns **one union AABB** for a *known* set. It does **not**
  answer this intent: a union box over scattered props is the whole scattered
  region, not the dense center — you need the **individual** positions to see
  where the density is. Complementary capability, different fix (set-scoped
  aggregate vs per-actor field projection), so filed separately per the board's
  "over-broad prior ticket must not absorb a distinct friction" rule.
- `F-actor-batch-tag-set` (OPEN, same family) — a tag **write** gap, unrelated
  data.
- `E-actor-list-no-limit-spills` (IN-REVIEW) — same method, added the `limit` +
  `fields` projection this ticket extends; it deliberately scoped the projection
  to `label/name/path/class`, so the *location* omission is a distinct follow-on
  gap, not a regression of that fix.
- `E-material-list-all-omits-position` (WONTFIX) — the material-namespace
  analogue (per-node `x`/`y` dropped from the list-all readback). Declined there
  as a low-value enrichment inferred from one clean task with zero perceived
  friction; the fix picker should weigh that precedent, but note the actor family
  keeps two OPEN Low siblings in exactly this profile
  (`F-actor-aggregate-bounding-box`, `F-actor-batch-tag-set`), and this ticket is
  explicitly framed as an enrichment/convenience gap, NOT a documented-contract
  violation (the misframing that partly sank the material ticket).

## Evidence (this task, outcome `clean` — no struggle, projection/batch gap only)

Task (focus `volume.add_cull_distance_volume`, namespace `volume`): optimize
render culling in an existing level by picking "a good anchor actor near the
center of that cluster" of scattered decorative props — the anchor was
deliberately left unspecified, so choosing it required reading the spatial layout.
The seed verb itself round-tripped clean (a `CullDistanceVolume` placed on the
`NaniteMesh` landmark, resized, and confirmed). The CallAnalyzer trace shows the
reconnaissance cost: to locate a landmark-plus-props cluster and pick an anchor
the agent issued **~13 individual `actor.get` transform reads** — 6 in
`/Game/Maps/Geometry/StaticMeshes` (`Basic_Asset`, `Basic_Asset2`,
`SM_Mesh_Sphere`, `SM_Mesh_UCX`, `SM_Mesh_GoodUVs`, `S_Spotlight`) then 7 in
`/Game/Maps/Geometry/Geometry_Nanite` (`NaniteMesh`, `Nanite_FallbackMesh`, three
Roman columns, `Cube`/`Cube4`/`Cube7`) — purely to triangulate a cluster center.
One bulk-location read (`actor.list {fields:["label","location"]}`) would have
collapsed each level's sampling into a single call. No error, retry, or wrong
result — the anchor was chosen correctly; this is a convenience/symmetry gap, not
a bug, and the agent voiced no friction about it. The open-ended "pick an anchor
near the center" phrasing makes the N-read reconnaissance inherent to the task,
which is exactly why it never surfaces as reported friction.

severity rationale: impact=soft-blocker-with-workaround (N per-actor `actor.get`
reads unioned client-side; here ~13 across two candidate levels) x reach=occasional
spatial-reconnaissance path -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean `volume.add_cull_distance_volume` task on `/Game/Maps/Geometry/Geometry_Nanite` (seed verb round-tripped clean; judge disposition clean, no defect). CallAnalyzer flagged ~13 single `actor.get` transform reads (6 in the StaticMeshes level, 7 in Geometry_Nanite) issued purely to triangulate a prop-cluster center and pick an anchor, because no method returns actor locations in bulk: `actor.get`/`get_transform`/`get_bounding_box` are single-actor, and `actor.list`'s `fields` projection (shipped by `E-actor-list-no-limit-spills #4`) allows only `label/name/path/class`, never `location`. Proposed: add `location`/`transform` to the `actor.list` projection allow-list (cheapest — the projection mechanism already exists), or a set-keyed batch transform getter. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — sibling `F-actor-aggregate-bounding-box` (same `actor-batch-set-asymmetry` family) returns a union AABB (does not answer per-actor-positions-to-find-a-cluster); `F-actor-batch-tag-set` is a tag write; `E-actor-list-no-*` are the class-filter and limit/spill gaps (this is the distinct location-projection follow-on); `E-material-list-all-omits-position` is the WONTFIX'd material analogue (weighed above). Framed as an enrichment/convenience gap, not a contract violation.
- `#2-rejected-includetransform-and-no-fields-key` `OPEN` WEAPONS-critic — Re-encountered in a WEAPONS critic review round 4, from a different intent than `#1` (reading bulk actor transforms during a weapons review, not triangulating a cluster centre), which is reach evidence rather than a new defect: `encounters` 1 → 2, **severity unchanged at Low, status unchanged at OPEN**. Two measurements that sharpen `#1`'s ask. First, the guessable spelling is **rejected outright**: `actor.list {"includeTransform": true}` returns `UNKNOWN_PARAMS`, so a caller who reaches for the obvious flag gets a hard error rather than a silently ignored key — which is the better of the two failure modes and worth preserving in whatever lands, but it does mean the first attempt always costs a call. Second, `#1`'s core finding is confirmed still true from the other direction: the `fields` projection carries **no transform key of any kind** — not `location`, not `transform`, not `bounds` — so neither the flag route nor the projection route reaches an actor's position, and bulk transforms still cost one `actor.get_transform` per actor. Note the flag spelling is a second reasonable guess alongside `#1`'s projection ask: a fixer implementing the `fields` allow-list extension should consider accepting `includeTransform` as an alias for `fields:["location","rotation","scale"]`, since that is what a caller tries first. No plugin source was opened for this entry and no `file:line` is claimed; both results are read off the responses.
