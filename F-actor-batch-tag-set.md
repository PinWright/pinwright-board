---
id: F-actor-batch-tag-set
title: "No one-call tagging of an actor set — add_tag is single-actor and spawn/spawn_batch take no tags param, so tagging a freshly-placed kit costs N calls"
status: OPEN
severity: Low
category: feature
tags: [actor-batch-set-asymmetry, add_tag, spawn_batch, batch, tags]
encounters: 1
lastSeen: 2026-07-11T08:32:47.0194806+03:00
---

# No batch/spawn-time tagging of an actor set

Tagging a freshly-spawned group of actors as one kit is a common blocking-out
intent ("tag every piece so I can find, select, and move them as a single unit
later"), but no method tags a set in one call:

- `actor.spawn` and `actor.spawn_batch` take **no `tags` param** (spawn_batch
  exposes a `folder` param but nothing for tags).
- `actor.add_tag` accepts only a single `actorName`.

So an N-actor kit needs N `add_tag` calls. This is **asymmetric with the
set-aware methods already in the same `actor` namespace**: `actor.set_folder`
takes an `actorNames[]` array, `actor.spawn_batch` takes a `folder`, and
`actor.find_by_tag` / `actor.delete_by_tag` already operate on the whole tagged
set at once. Only the tag-**write** verb is single-actor.

## What it should do

Either of:
- add a `tags` (string array) param to `actor.spawn` / `actor.spawn_batch` so a
  set can be tagged at spawn time (mirrors the existing `folder` param on
  spawn_batch, and the sibling `F-actor-aggregate-bounding-box`'s ask that the
  spawn/set verbs grow set-first-class params), or
- accept an `actorNames[]` (or a `tag`/`folder` selector) on `actor.add_tag` so
  one call tags many, mirroring `actor.set_folder`'s `actorNames[]`.

## Evidence (this task, outcome `clean` — no struggle, batch-capability gap only)

Task: block out a combat arena (floor + 4 corner pillars + center pedestal) and
"tag every piece as one kit." The agent first did capability-gap discipline —
grepped `actor.*.md` for `tag` (found only add_tag / remove_tag / find_by_tag /
delete_by_tag, no batch-tag verb) and re-grepped `actor.spawn_batch.md` for a
`tags` param (none) — then, per its own note, "No batch-tag helper exists and
spawn takes no tags param, so per-actor actor.add_tag is the intended path."
It then issued **6 separate `actor.add_tag` calls**, one per kit actor
(`Arena_Pillar_NE`/`NW`/`SE`/`SW`, `Arena_Floor`, `Arena_Pedestal`), all
`tag=ArenaKit`, each returning ok. One `tags`-aware spawn (or one array-aware
add_tag) would have replaced all 6. No error, retry, or wrong result occurred —
the agent tagged all 6 cleanly; this is a convenience/symmetry gap, not a bug.

severity rationale: impact=soft-blocker-with-workaround (N clean per-actor calls, here 6-for-1) x reach=occasional set-authoring path -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean arena block-out task (`ExampleProjectWelcome`). CallAnalyzer flagged 6 `actor.add_tag` calls (one per kit actor, all `tag=ArenaKit`) as a batch-capability gap: no `tags` param on `actor.spawn`/`actor.spawn_batch` and no `actorNames[]` on `actor.add_tag`, asymmetric with the set-aware `actor.set_folder`(`actorNames[]`) / `actor.find_by_tag` / `actor.delete_by_tag` / `actor.spawn_batch`(`folder`). Judge disposition: clean signal, no replay — behavior is correct, purely a convenience gap. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX found no batch-tag ticket; `F-inspect-list-actor-tags` is tag **enumeration** (a census read), a different capability. Sibling `F-actor-aggregate-bounding-box` shares the `actor-batch-set-asymmetry` family (single-actor verb where a set-aware one should exist) but is a distinct capability (bbox read).
