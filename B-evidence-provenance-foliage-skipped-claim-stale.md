---
id: B-evidence-provenance-foliage-skipped-claim-stale
title: "level-review.evidence-and-provenance.md:84 says actor.spawn_batch and foliage.add_instances drop malformed entries silently and tells reviewers to compare counts; both now report skipped[]/skippedCount, and only actor.spawn_batch places a location-less entry at the world origin"
status: OPEN
severity: Low
rice: [1, 2, 1, 1]
priority: 17
category: bug
tags: [docs, wiki-src, level-review, evidence-and-provenance, foliage, add_instances, spawn_batch, paint, skipped, world-origin, stale-doc, duplicated-fact]
---

# Review checklist line 84 is stale about skipped entries

`docs/wiki-src/level-review.evidence-and-provenance.md:84` reads:

> - `actor.spawn_batch` and `foliage.add_instances` drop malformed entries silently — compare counts,
>   and note that an entry missing `location` is not dropped at all: it spawns at the world origin.

Current behaviour at 7230b41d:

- `foliage.add_instances` reports every unusable entry in `skipped[]` (`{index, reason}`) with an
  always-present `skippedCount` (registration text
  `Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:1858`; emitted `:2151-2154`), and
  **refuses** an entry with no location (`:1954`) instead of placing it at the origin.
- `actor.spawn_batch` reports `skipped[]`/`skippedCount` the same way
  (`Source/PinWright/Private/Handlers/Actor/SpawnBatchHandler.cpp:34`, emitted `:364-368`), and
  **does** spawn an entry that omits `location` at the world origin (param doc `:40`).
- `foliage.paint` also reports non-object and coordinate-less `locations[]` entries in `skipped[]`
  rather than placing them at the origin (`FoliageHandler.cpp:786-795`, emitted `:1213-1217`).

So the "silently" half is false for both named verbs, the world-origin half is true only for
`actor.spawn_batch`, and "compare counts" prescribes a weaker check than the per-entry reasons the
responses already carry. The correct statement already exists in `docs/wiki-src/level-building.md:97`
and `docs/wiki-src/actor.md:345`; line 84 is a drifted copy.

**Fix:** replace `:84` in `docs/wiki-src/level-review.evidence-and-provenance.md` (the source page,
not the generated `Saved/PinWright/wiki/` copy) with a line that reads the fields and references the
canonical page, e.g.:

    - `actor.spawn_batch`, `foliage.add_instances` and `foliage.paint` report every unusable entry
      in `skipped[]` with a reason and always publish `skippedCount` — read those rather than
      comparing totals. Only `actor.spawn_batch` accepts an entry with no `location`, and spawns
      it at the world origin (see `level-building.md`).

**Acceptance:** the regenerated `level-review.evidence-and-provenance` page no longer says "drop
malformed entries silently" or "compare counts", names `skipped[]`/`skippedCount`, and attributes
the world-origin behaviour to `actor.spawn_batch` only; `grep -rn "drop malformed entries silently"
docs/wiki-src` returns nothing.

severity rationale: impact=Low — documentation only; the harm is a reviewer doing a weaker check and
distrusting two correct fields × reach=normal (a checklist page read once per review) -> Low

## History
- `#1-stale-in-three-of-four-claims` `OPEN` reporter — Source-read only, editor not running; no RPC was called. `Docs/wiki-src/level-review.evidence-and-provenance.md:84` makes four claims and one holds. Both "drops malformed entries silently" halves are false: `foliage.add_instances` reports `skipped[]` via `NoteSkipped` (`FoliageHandler.cpp:695`, called `:710`/`:715`/`:736`/`:801`/`:826`) with an always-present `skippedCount` (`:906`, array `:908-909`) and says so in its own registration text (`:640`); `actor.spawn_batch` does the same (`SpawnBatchHandler.cpp:34` registration, `:364`, `:367-368`). The world-origin half is false for `foliage.add_instances`, which refuses a location-less entry outright (`FoliageHandler.cpp:736-737`), and true for `actor.spawn_batch`, whose own param doc states it (`SpawnBatchHandler.cpp:40`) — so the coordinator's instruction to check the `spawn_batch` half of the sentence was worth following: fixing only the foliage half would have left half a wrong sentence standing. Two other wiki-src pages already carry the correct text — `level-building.md:97` and `actor.md:320` — so `:84` is a duplicated fact that drifted while its siblings stayed right, which is the case for referencing rather than restating. Separately, the sentence misses the verb that still does both things: `foliage.paint` drops non-object `locations[]` entries with no record (`FoliageHandler.cpp:126-137`) and turns a location-less object into `FVector(0,0,0)` (`:130-134`, `:145-148`), filed as `B-foliage-paint-does-no-ground-projection` — so the fix is a rewrite, not a deletion. Fix goes in `Docs/wiki-src/` only; the `Saved/PinWright/wiki/` copy is regenerated at launch and has already drifted out of line alignment (its own line 84 is a different bullet), confirmed by diff. Dedup: searched the board for `evidence-and-provenance`, `level-review`, `spawn_batch` + docs, `world origin`, `skipped`, and every `B-*doc*` / `E-*doc*` ticket. `B-animation-provenance-misclassified` is the only provenance-named file and concerns animation asset dumps. Nothing covers this page or this line.
- `#2-rephrased` `OPEN` developer — The doc defect at `level-review.evidence-and-provenance.md:84` is unchanged, but the old body's proposed replacement text was wrong: it said `foliage.paint` drops non-object entries with no record and places a location-less entry at the origin, while `foliage.paint` now reports both in `skipped[]` (`FoliageHandler.cpp:786-795`; `B-foliage-paint-does-no-ground-projection` DONE). Replacement text now says all three verbs report `skipped[]` and only `actor.spawn_batch` spawns at the origin; refreshed citations (`FoliageHandler.cpp:1858`, `:1954`, `:2151-2154`; `actor.md:345`), dropped the foliage.paint section and the "not RPC-verified" note, added Acceptance. Severity unchanged (Low).
