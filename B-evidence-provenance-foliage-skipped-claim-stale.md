---
id: B-evidence-provenance-foliage-skipped-claim-stale
title: "level-review.evidence-and-provenance.md:84 is wrong in three of its four claims — both named verbs now report skipped[]/skippedCount, and foliage.add_instances refuses a location-less entry rather than placing it at the world origin; the verb that really does both, foliage.paint, is not mentioned, and level-building.md:97 already carries the correct text"
status: OPEN
severity: Low
category: bug
tags: [docs, wiki-src, level-review, evidence-and-provenance, foliage, add_instances, spawn_batch, paint, skipped, world-origin, stale-doc, duplicated-fact]
---

# A review checklist that sends the reviewer to a weaker check on two verbs, and names none of the one that is still broken

`Docs/wiki-src/level-review.evidence-and-provenance.md:84`, verbatim:

> - `actor.spawn_batch` and `foliage.add_instances` drop malformed entries silently — compare counts,
>   and note that an entry missing `location` is not dropped at all: it spawns at the world origin.

Four claims. One holds.

**1. "`foliage.add_instances` drops malformed entries silently" — FALSE.** Its registration text
already promises otherwise: *"Entries the parser cannot use are reported in `skipped[]`
({index, reason}, capped) with the true total in `skippedCount` - they are never dropped silently"*
(`Handlers/Environment/FoliageHandler.cpp:640`). Implemented by `NoteSkipped`
(`:695`), called at `:710`, `:715`, `:736`, `:801`, `:826`; emitted as an always-present
`skippedCount` (`:906`) plus `skipped` and `skippedTruncated` when non-zero (`:908-909`).

**2. "`actor.spawn_batch` drops malformed entries silently" — FALSE, same way.** Its registration
carries the identical promise (`Handlers/Actor/SpawnBatchHandler.cpp:34`), and `skippedCount` is
always present with `skipped` / `skippedTruncated` alongside (`:364`, `:367-368`).

**3. "an entry missing `location` … spawns at the world origin" — FALSE for `foliage.add_instances`.**
It refuses the entry: `NoteSkipped(EntryIndex, TEXT("transforms[] entry has no valid location;
supply an object {x,y,z} or an array [x,y,z]"))` followed by `continue` (`FoliageHandler.cpp:736-737`).

**4. Same claim — TRUE for `actor.spawn_batch`.** Its own parameter doc says so: *"a missing
`location` defaults to the world origin"* (`SpawnBatchHandler.cpp:40`).

## The advice it gives is also worse than what is available

*"compare counts"* was the right instrument when nothing was reported. Both verbs now hand back
per-entry `{index, reason}` rows. A reviewer following this line does arithmetic on totals and gets
no reason for any drop, when the response already names each one — a strictly weaker check than the
tool offers, prescribed by the page whose whole subject is choosing the right instrument.

## The fact is already documented correctly, twice, somewhere else

- `Docs/wiki-src/level-building.md:97` — *"A non-object entry is skipped and reported in `skipped[]`,
  but an object that omits `location` is valid and spawns at the **world origin** — it is not
  dropped. Validate locations before the call, then check `count + skippedCount == requested`; counts
  alone cannot detect an accidental origin placement."* Accurate and current.
- `Docs/wiki-src/actor.md:320` — *"An entry without `location` spawns at world origin; it is *not*
  dropped."* Also accurate.

So `:84` is a **duplicated fact that drifted**, which is exactly the failure the project's own
documentation rule exists to prevent (*"Do not restate the request, repeat information, or copy
another document; reference its canonical path instead"*). The two copies that were maintained stayed
right; the third copy, in a different page, went stale and now contradicts them.

## And it misses the verb that still does both things

`foliage.paint` **does** drop malformed entries silently and **does** place a location-less entry at
the world origin — the two behaviours this line attributes to the two verbs that no longer have them:

- a `locations[]` element that is not a JSON object is skipped with no `skipped[]`, no
  `skippedCount`, and no reason (`FoliageHandler.cpp:126-137`);
- an object with no `x`/`y`/`z` becomes `FVector(0,0,0)`, because the `double X = 0, Y = 0, Z = 0`
  initialisers survive three failed `TryGetNumberField` calls (`:130-134`, and the same shape in the
  single-`position` branch at `:145-148`).

Filed as `B-foliage-paint-does-no-ground-projection`. A doc fix that only corrects the two stale
verbs leaves the reviewer with no warning about the one that is still live, which is why this ticket
asks for a rewrite rather than a deletion.

## Fix

Replace `:84` with a line that references rather than restates, and that names the real case:

    - `actor.spawn_batch` and `foliage.add_instances` report every unusable entry in `skipped[]`
      with a reason, and always publish `skippedCount` — read those, do not compare totals.
      `actor.spawn_batch` still accepts an entry with no `location` and spawns it at the world
      origin (see `level-building.md` § spawn_batch); `foliage.add_instances` refuses it.
      `foliage.paint` does neither: it drops non-object entries with no record and places a
      location-less entry at the origin.

Edit `Docs/wiki-src/level-review.evidence-and-provenance.md` **only**. The copy under
`Saved/PinWright/wiki/` is regenerated at editor launch by `WikiDiskGenerator` and has already
drifted out of line alignment with the source (its line 84 is a different bullet entirely), so
editing it would patch the output and leave the generator's input wrong — the process norm this
board records as *"Fix the process, not the output"*.

Worth doing in the same pass: grep the other `wiki-src` pages for further restatements of the same
fact. Two were found and are both correct; a third that went stale is evidence the fact is being
copied faster than it is being checked.

## Not RPC-verified

Source-read only; the editor was not running and no `actor.spawn_batch`, `foliage.add_instances` or
`foliage.paint` call was made. Every claim above is from the handler source and the doc text, both of
which a reviewer can open. Nothing here needs an editor to settle.

severity rationale: impact=Low — documentation: no verb behaves wrongly because of it, and the concrete harm is a reviewer performing a weaker check (count arithmetic) than the response supports, plus distrusting two fields that are correct × reach=normal — `level-review.evidence-and-provenance` is a checklist page read once per review rather than a method that runs every session, so the every-session bump does not apply; the case for Medium is that a *review* doc is a safety instrument and a wrong instrument is worse than a missing one, and it is declined because the failure costs verification effort rather than producing a wrong result — with the caveat that the same argument gets stronger, not weaker, the longer `foliage.paint` goes unmentioned -> Low

## History
- `#1-stale-in-three-of-four-claims` `OPEN` reporter — Source-read only, editor not running; no RPC was called. `Docs/wiki-src/level-review.evidence-and-provenance.md:84` makes four claims and one holds. Both "drops malformed entries silently" halves are false: `foliage.add_instances` reports `skipped[]` via `NoteSkipped` (`FoliageHandler.cpp:695`, called `:710`/`:715`/`:736`/`:801`/`:826`) with an always-present `skippedCount` (`:906`, array `:908-909`) and says so in its own registration text (`:640`); `actor.spawn_batch` does the same (`SpawnBatchHandler.cpp:34` registration, `:364`, `:367-368`). The world-origin half is false for `foliage.add_instances`, which refuses a location-less entry outright (`FoliageHandler.cpp:736-737`), and true for `actor.spawn_batch`, whose own param doc states it (`SpawnBatchHandler.cpp:40`) — so the coordinator's instruction to check the `spawn_batch` half of the sentence was worth following: fixing only the foliage half would have left half a wrong sentence standing. Two other wiki-src pages already carry the correct text — `level-building.md:97` and `actor.md:320` — so `:84` is a duplicated fact that drifted while its siblings stayed right, which is the case for referencing rather than restating. Separately, the sentence misses the verb that still does both things: `foliage.paint` drops non-object `locations[]` entries with no record (`FoliageHandler.cpp:126-137`) and turns a location-less object into `FVector(0,0,0)` (`:130-134`, `:145-148`), filed as `B-foliage-paint-does-no-ground-projection` — so the fix is a rewrite, not a deletion. Fix goes in `Docs/wiki-src/` only; the `Saved/PinWright/wiki/` copy is regenerated at launch and has already drifted out of line alignment (its own line 84 is a different bullet), confirmed by diff. Dedup: searched the board for `evidence-and-provenance`, `level-review`, `spawn_batch` + docs, `world origin`, `skipped`, and every `B-*doc*` / `E-*doc*` ticket. `B-animation-provenance-misclassified` is the only provenance-named file and concerns animation asset dumps. Nothing covers this page or this line.
