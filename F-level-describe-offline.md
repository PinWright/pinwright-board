---
id: F-level-describe-offline
title: "No read-only verb exposes a level's actor labels or transforms without loading the world — under a held world lock, 24 actors had to be recovered by hand-parsing length-prefixed FStrings out of the .umap bytes"
status: OPEN
severity: Medium
category: feature
tags: [level, actor, read-only, offline, umap, world-lock, asset-dump, level-describe, package-parse, weapons]
encounters: 1
costly: 1
lastSeen: 2026-09-03T00:00:00Z
---

# "What actors are in this map, and where?" has no answer that does not take the world

Every published route to a level's contents goes through the active editor world. When another
stream holds the world lock — the normal condition when several agents share one editor — the
question is unanswerable through the plugin, for a map that is sitting on disk in a readable format.

## The three routes, and why each is closed

**1. `asset.dump` on a `.umap` is a world-loading background job.** It does produce the answer
(`actors/manifest.json` plus per-actor JSON in the `pinwright.actor-describe.v1` shape), but it gets
there by loading the world — so it is exactly the thing an asset-only agent under a held lock must
not run, and it is asynchronous besides.

**2. `actor.*` needs the level to be the active world.** `actor.list`, `actor.describe`,
`actor.get_transform` all read the active editor world. There is no `levelPath` that redirects them
at an unloaded map.

**3. `system.inspect.*` needs the same.** Same precondition, same wall.

`level.get_info` / `get_actors` / `get_bounds` do take a `levelPath`, and they are the closest
existing shape — but they resolve it only against levels **already loaded into the active world**,
which is the subject of `E-level-getters-require-loaded-not-on-disk`.

## What that cost — measured

Recovering **24 actor labels and world locations** from
`X:\src\unreal\EAContentExamples58\Content\FPS\Test\T_Weapons.umap` required reading the package
file directly and **hand-parsing length-prefixed FStrings and transform triples out of the raw
bytes** — reconstructing, by hand and without a schema, data the plugin already knows how to emit in
a documented shape.

That is not a workaround in the rubric's sense (a documented alternative, or many extra calls). It
is a bespoke binary parse of an engine format, with no validation, that happens to have worked once.
Anything it got subtly wrong would have been invisible.

## What is asked for

A **`level.describe_offline`** (name illustrative) that reads a `.umap` from disk and returns its
actors without loading it or touching the active world:

- **Minimum:** per actor `{label, name, class, location, rotation, scale}` — the fields that make
  "did the layout land, and where is everything" answerable. This is what the hand-parse was after.
- **Useful next:** `folder`, `tags`, `guid`, and the actor count as a summary line; plus the
  external-actor references that One File Per Actor maps split out, listed rather than resolved
  (the `actors/manifest.json` shape already distinguishes embedded from external).
- **Contract that makes it worth having:** *never* loads the world, *never* changes the active level,
  and says so in its own documentation — so an asset-only agent can call it under a held lock as a
  matter of policy, not of hope. That guarantee is most of the value; a verb that "usually" avoids
  loading is not usable here.
- **Shape reuse:** emit `pinwright.actor-describe.v1`, the schema `asset.dump`'s `actors/*.json`
  already uses, so offline and loaded reads are diffable against each other without translation.

## Why this is not the same ask that was already declined

`E-level-getters-require-loaded-not-on-disk` (IN-REVIEW) explicitly ruled the offline path out of
scope — verbatim: *"auto-loading / peeking an unloaded on-disk map for a read getter (load-then-restore
or a registry peek …) is intentionally out of scope: it drags active-world mutation/restore risk into
a read-only path for a marginal payoff"*. **That reasoning is correct and this ticket does not
contest it.** It is an argument against making `level.get_info` load a map behind the caller's back —
against a *hidden* load inside an existing getter. It is not an argument against a separate,
explicitly-named verb that parses the package file and never loads anything: a disk parse cannot
mutate the world it never opens, so the risk that justified the exclusion does not exist on this
path. The payoff is also no longer marginal, since the alternative measured here was a hand-written
binary parse.

## Second-order value

An offline reader lets **asset-only agents prepare world work safely** — enumerate what a map
contains, plan edits, and validate a plan's assumptions — all before anyone takes the world lock.
Today that preparation either waits for the lock or does not happen, which is what pushes agents into
exactly the improvisation this ticket documents.

## Severity

**Medium** — the rubric's hard-blocker band, adjusted for reach. Through the published surface the
task is impossible while the lock is held: there is no documented workaround and no many-extra-calls
route, only an off-surface binary parse. Not High because it is not an every-session path — it bites
specifically when the world lock is contended, which is common in multi-agent sessions and absent in
single-agent ones.

## Related

- `E-level-getters-require-loaded-not-on-disk` (IN-REVIEW, Low) — the nearest existing surface, and
  the ticket that declared the offline peek out of scope for those three getters. Its fix
  (`LEVEL_NOT_LOADED` instead of a misleading `LEVEL_NOT_FOUND`) makes the wall *honest*; this ticket
  asks for a door. They are complements, not alternatives, and the new verb is the natural thing for
  that improved error message to point at.
- `B-level-load-dirty-world-memory-leak-fatal`, `B-level-load-dirty-world-fatal` — the cost of the
  load this verb avoids, and part of why "just load it" is not a neutral suggestion.
- `E-actor-list-no-limit-spills`, `E-actor-describe-no-header-only-read` (IN-REVIEW) — the loaded-world
  actor reads, and their response-size ergonomics. Whatever projection they settle on should apply
  here too; a 24-actor map is small, a real level is not.
- `B-dump-folder-includes-level-subobjects`, `B-asset-dump-folder-no-completion-signal` — the
  `asset.dump` route's own problems, which is the other reason it is a poor substitute.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 2. No read-only verb exposes a level's actor labels or transforms without loading the world: `asset.dump` on a `.umap` is a world-loading background job (it does emit `actors/manifest.json` plus per-actor `pinwright.actor-describe.v1` files, but only by loading the world), and `actor.*` / `system.inspect.*` all require the level to be the active world; `level.get_info` / `get_actors` / `get_bounds` take a `levelPath` but resolve it only against levels already loaded, per `E-level-getters-require-loaded-not-on-disk`. With the world lock held by another stream, recovering 24 actor labels and world locations from `X:\src\unreal\EAContentExamples58\Content\FPS\Test\T_Weapons.umap` required reading the package file and hand-parsing length-prefixed FStrings and transform triples out of the raw bytes — a bespoke, unvalidated binary parse of an engine format, reconstructing data the plugin already emits in a documented shape. Ask: a `level.describe_offline`-style verb that reads a `.umap` from disk without loading it — minimum `{label, name, class, location, rotation, scale}` per actor, then `folder`/`tags`/`guid` and external-actor references listed rather than resolved, emitting the existing `pinwright.actor-describe.v1` schema so offline and loaded reads are diffable, and carrying a documented guarantee that it never loads the world or changes the active level (that guarantee is most of the value — a verb that only usually avoids loading is unusable under a lock). Explicitly not the ask `E-level-getters-require-loaded-not-on-disk` declined: that ticket ruled out a *hidden* load-then-restore or registry peek inside an existing getter because it "drags active-world mutation/restore risk into a read-only path" — correct, and untouched here, since a disk parse cannot mutate a world it never opens; the payoff is also no longer marginal now that the measured alternative is a hand-written binary parse. Second-order value: asset-only agents could enumerate and validate a map before anyone takes the world lock, which is the preparation that today either waits or turns into the improvisation above. Severity Medium on the hard-blocker band (no documented workaround, no many-calls route, only an off-surface parse), not High because the block is specific to a contended world lock rather than every session.
