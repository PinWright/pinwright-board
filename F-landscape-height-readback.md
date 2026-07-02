---
id: F-landscape-height-readback
title: "No landscape height/region readback verb — landscape.sculpt/edit can WRITE heights but nothing samples them, so a sculpt round-trip can only be verified by emulating a readback through internal component fields (all of which come back stale or NOT_FOUND)"
status: OPEN
severity: Medium
category: feature
tags: [write-verb-no-content-readback, landscape, sculpt, edit, readback, verify, round-trip, get_heights, sample_region]
encounters: 1
lastSeen: 2026-07-02T09:33:37+0000
---

# The landscape height-editing verbs have no content readback — you cannot sample the terrain to confirm a sculpt landed

The `landscape.*` namespace has verbs that **write the heightmap** —
`landscape.sculpt` (single Raise/Lower/Flatten stamp) and `landscape.edit`
(bulk `set`/flatten over a region) — but **no read counterpart**. The full
namespace registers only `create`, `create_grass_type`,
`create_procedural_terrain`, `edit`, `sculpt`, `set_material` (confirmed in the
audited transcript's namespace walk and an index-wide wiki Grep for
`height|Bounds|readback|get_height|sample|elevation`, which surfaced no
landscape height-sample method). `landscape.edit` only WRITES
(`operation='set'`/flatten) — it cannot read a region back.

So after sculpting a hill, ridge, and building pad — the audited task's exact
intent, whose success criterion is "visibly non-flat relief" — there is **no
first-class way to sample the current heights or a region's min/max/mean Z** and
confirm the terrain actually took the shape that was written. The sculpt calls
return `success:true` with a `modifiedVertices` count, which proves the write was
not a no-op, but that is a *write-side* signal: it says "N vertices were touched,"
not "the central region is now higher than the flat baseline." The only way to
close that loop is a height sample the namespace does not offer.

## Why this is a distinct capability gap (not the already-filed bounds bug)

The judge filed `B-landscape-sculpt-stale-bounds` — a **bug**: `sculpt`/`edit`
write + `Flush` but never refresh the component `CachedLocalBox`/collision, so
every bounds readback (`actor.get_component_property CachedLocalBox` AND the live
`actor.get_bounding_box`) reports Z=0 after a real raise. That ticket's fix
(refresh bounds after the height write) restores the **actor's overall bounding
box**.

This ticket is a different axis and **survives that fix**:

1. A refreshed bounding box gives one number — the actor's total Z extent. It
   does **not** let a caller confirm that *the central hill* rose while *the
   building pad* stayed flat while *the secondary ridge* rose less — the
   per-region relief the task actually asked for. Only a height/region sample
   answers that.
2. Even with correct bounds, verifying "region A is higher than region B" or
   "the pad at the near edge is flat" requires reading heights at coordinates,
   which no verb exposes. Bounds are a whole-actor aggregate; this is a
   spatial-sampling capability.

This mirrors the accepted precedent `F-texture-pixel-stats-readback` (IN-REVIEW):
the pixel-mutating `texture.*` verbs (`desaturate`/`invert`/`adjust_levels`) had
no *content* readback (only metadata), so a transform could only be inferred, and
a `texture.get_pixel_stats` feature was filed **separately** from the
metadata-correctness bug the judge filed on that task. Same shape here: a
write-family with no content read, filed as a feature distinct from the
derived-state bug.

## What it should do

Add a live landscape height readback so the sculpt/edit round-trip closes, e.g.
`landscape.get_heights` / `landscape.sample_region` (or an `operation='get'` on
`landscape.edit`) that returns, for a heightmap-pixel region, either the raw
`uint16` height samples or cheap region aggregates (`minZ`/`maxZ`/`meanZ`, and a
world-space Z for min/max). That is readable via the same
`FLandscapeEditDataInterface::GetHeightData` the write path already uses (the
mutator reads-modifies-writes the region), so no new access mechanism is needed.
With a region min/max/mean, a caller can confirm "central region raised above
baseline," "pad region flattened to a constant Z," and "ridge region raised less
than the hill" — turning `modifiedVertices` (a write-side count) into an actual
shape verification.

**Workaround:** none that samples heights. The audited task fell back to reading
internal component fields via `actor.get_component_property`
(`CachedLocalBox.Max.Z`, full `CachedLocalBox`, `Bounds`, and the
heightfield-collision component's `CachedLocalBox`) — 4 calls plus 2 discovery
wiki reads — and got stale `0` or `[NOT_FOUND]` every time, never a live
post-sculpt height. It ultimately trusted the ordered `modifiedVertices` counts
(the success check's permitted alternative), i.e. it could not directly verify
relief at all.

## Friction evidence (this task — `landscape` prototype-hillside sculpt, 22 calls, outcome tool_bug/done)

Story: create a small landscape near origin, raise a central hill + a secondary
ridge, flatten a building pad, and make a grass-type asset. The terrain edits all
succeeded (5 Raise stamps `modifiedVertices` 3969/1681/841/841/841, 1 Flatten
729). To confirm the core "measurably higher" criterion the agent then spent 4
`actor.get_component_property` calls emulating a readback — `LandscapeComponent_0
CachedLocalBox.Max.Z` → `0`, full `CachedLocalBox` → `Max:[63,63,0]`,
`LandscapeComponent_0 Bounds` → `[NOT_FOUND] Property 'Bounds' not found`,
`HeightfieldCollision_0 CachedLocalBox` → `Max` Z `0` — plus 2 wiki discovery
reads, none of which yielded a fresh height. The auditee's note: *"the actor's
Z-bounds readback ... stayed at stale Max.Z=0 after sculpting, so it could not
confirm relief directly ... no landscape height/region readback verb exists."*
No retries or `python.execute` fallback — the gap is the *unverifiability* of the
sculpt result, not a failure. (The stale-`0` bounds themselves are the judge's
`B-landscape-sculpt-stale-bounds`; the `Bounds` `NOT_FOUND` is an expected
transient-field limit; this ticket is solely the missing height-sample verb.)

severity rationale: impact=soft-blocker (a normal terrain-verify intent is doable only via a multi-call source-dive emulation that still fails to yield the value) × reach=every-landscape-session (sculpt/edit are the core terrain-shaping verbs and their result is normally verified) -> Medium

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `landscape` prototype-hillside task (22 calls, outcome tool_bug; judge filed `B-landscape-sculpt-stale-bounds`, the missing bounds/collision refresh). Distinct PROCESS/capability angle: the landscape write verbs (`sculpt`, `edit`) have NO height/region readback counterpart — the namespace registers only create/create_grass_type/create_procedural_terrain/edit/sculpt/set_material, and an index-wide wiki Grep for height/sample/readback surfaced none. So after sculpting a hill/ridge/pad the agent could not sample the terrain to confirm relief and burned 4 `actor.get_component_property` emulation calls (CachedLocalBox Max.Z=0, full CachedLocalBox Max:[63,63,0], `Bounds` [NOT_FOUND], collision CachedLocalBox Max Z=0) + 2 discovery reads, none yielding a live height, then fell back to trusting `modifiedVertices` counts. This survives the judge's bounds-refresh fix: a whole-actor bounding box is one aggregate number and cannot confirm per-region relief (hill up, pad flat, ridge up-less), which is what the task asked. Proposed: add `landscape.get_heights` / `landscape.sample_region` (or `operation='get'` on `landscape.edit`) returning region uint16 samples or min/max/mean Z via the same `FLandscapeEditDataInterface::GetHeightData` the write path already uses. Precedent: `F-texture-pixel-stats-readback` (IN-REVIEW) — same shape (pixel-mutating verbs, no content readback → new stats verb, filed separately from the metadata bug). Dedup: ripgrep across OPEN+closed — no landscape height/sample/region-readback ticket exists; the landscape files are `B-landscape-sculpt-stale-bounds` (bounds-refresh bug — different axis), `B-landscape-create-*` (create geometry), `B-landscape-handler-bypasses-ctx`, `E-landscape-edit-extent-error-not-diagnostic` (error text), `E-generate-lods-landscapepath-misnomer` (unrelated param naming). Family tag `write-verb-no-content-readback` shared with `F-texture-pixel-stats-readback`.
