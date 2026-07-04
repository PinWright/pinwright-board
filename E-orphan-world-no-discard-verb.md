---
id: E-orphan-world-no-discard-verb
title: "An orphaned in-memory world (e.g. from level.create) blocks re-creation with LEVEL_ALREADY_EXISTS and has no discard verb — only escape is loading an unrelated map to force GC"
status: OPEN
severity: Low
category: ergonomic
tags: [in-memory-orphan-world, level, create, orphan, discard, gc]
encounters: 1
lastSeen: 2026-07-04T16:30:08.0157490+03:00
---

# No verb to discard an orphaned in-memory world — the only cleanup is loading a decoy map

When `level.create` (and other creators) build a world that is never persisted,
they leave a live in-memory `UWorld` registered at the target `/Game/` path. That
orphan then **blocks re-creating a level at the same path**: the next
`level.structure.create_level` on that name fails

> `[LEVEL_ALREADY_EXISTS] Level already exists in memory: /Game/Maps/LightingSandbox. Use load_level or provide a different name.`

The collision error's suggested recovery is a dead end for this state:

- `load_level` on the phantom path fails — there is nothing on disk
  (`FILE_NOT_FOUND` / `LEVEL_NOT_PERSISTED`, the subject of
  `E-level-load-file-not-found-vs-in-memory-orphan`), so the "Use load_level"
  advice loops back to a verb that cannot succeed for an unsaved phantom.
- "provide a different name" only sidesteps the collision; it does not clear the
  orphan, and the new name strands its own phantom.

There is **no discard / reset-active-world verb** to drop the orphan. The only
working escape is to `level.load` an **unrelated, real on-disk map** purely to
trigger GC of the phantom — an obscure, undiscoverable side effect. In the
audited task the agent had to do this **twice**: once to evict the
`/Game/Maps/LightingSandbox` phantom and again to evict the `/Game/Maps/LS_Tmp`
phantom, each time side-loading `/Game/Maps/ExampleProjectWelcome` for no reason
other than to reclaim the orphan before retrying.

## Why this is a distinct (process) angle

- `B-create-level-saved-true-no-umap` (IN-REVIEW) tears down the orphan **only on
  `level.structure.create_level`'s own save-failure path**, so a retry of *that*
  verb isn't blocked. It does not clear a phantom left by a prior `level.create`
  (a different handler) — the collision in this trace came from `create_level`
  detecting `level.create`'s phantom, which the B- teardown never touches.
- `B-level-create-makes-wp-map` (IN-REVIEW) fixes what kind of world
  `level.create` builds (WP vs flat); it does not add a cleanup path for the
  orphan it strands.
- `E-level-load-file-not-found-vs-in-memory-orphan` (IN-REVIEW) fixes the
  `level.load` **error wording** for an in-memory orphan; its `#2` explicitly
  confirms `level.create` "still strands an unsaved in-memory UWorld on SaveMap
  failure (emits SAVE_FAILED with no teardown)" — i.e. the phantom producer
  survives the other fixes — but that ticket offers no verb to remove it.
- `E-create-level-save-false-world-not-active` (IN-REVIEW) discussed a dedicated
  *activation* verb and marked it out of scope; a *discard* verb is the orthogonal
  missing piece.

So even with the whole persistence cluster fixed, an agent that abandons a
freshly-created-but-unsaved world still has no clean way to drop it, and the
`LEVEL_ALREADY_EXISTS` collision still points at broken recovery advice.

## What it should do

- Provide a discard / reset verb (e.g. `level.discard` / `level.new_active` / a
  `resetActiveWorld` flag) that drops an orphaned in-memory world without
  side-loading an unrelated map, so cleanup is a first-class operation rather than
  a GC side effect.
- And/or make the `LEVEL_ALREADY_EXISTS` message name the real recovery for the
  in-memory-only case (the world exists only in memory and was never saved —
  discard it or save it) instead of pointing at `load_level`, which fails on an
  unsaved phantom.

## Evidence (from the audited lighting-sandbox task, call_count=29 RPCs)

- `level.create {levelName:LightingSandbox}` → ok, becomes active world (11 actors)
  → spawn lights → saves all fail (see the persistence cluster) → retry
  `level.structure.create_level {LightingSandbox, save:true}` →
  `[LEVEL_ALREADY_EXISTS] Level already exists in memory: /Game/Maps/LightingSandbox`.
- `level.load {/Game/Maps/ExampleProjectWelcome}` — issued purely to GC-evict the
  LightingSandbox phantom before retrying.
- Later, the temp-name attempt repeated the whole dance:
  `level.create {LS_Tmp}` → spawn lights → `save_as` fails →
  `level.load {/Game/Maps/ExampleProjectWelcome}` — a second decoy load, purely to
  evict the LS_Tmp phantom.
- Net: 2 `level.load` calls on an unrelated map spent entirely on phantom
  eviction, with no cleanup verb available and the collision error's `load_level`
  advice non-functional for the in-memory-only state.

severity rationale: impact=soft-blocker-with-obscure-undocumented-workaround (Medium) × reach=rare (only on re-create-after-orphan recovery) -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a lighting-sandbox setup task (create /Game/Maps/LightingSandbox with a directional + point light, save, back up, delete backup). The run was blocked by the level-persistence cluster (filed/updated under `B-create-level-saved-true-no-umap` `#5`, `B-lighting-create-level-false-success-no-umap`, `B-level-save-saved-true-in-memory-no-umap`), but a distinct PROCESS cost surfaced on top: `level.create` left an unpersistable in-memory phantom at `/Game/Maps/LightingSandbox`, so the follow-up `level.structure.create_level {LightingSandbox, save:true}` hit `[LEVEL_ALREADY_EXISTS] Level already exists in memory: /Game/Maps/LightingSandbox. Use load_level or provide a different name.` — but `load_level` on that path fails (nothing on disk) and no discard verb exists, so the agent side-loaded `/Game/Maps/ExampleProjectWelcome` purely to GC the orphan, then repeated the identical eviction for a second `LS_Tmp` phantom. Two decoy-map loads spent on cleanup that a discard/reset verb (or an accurate collision message) would eliminate. Dedup: distinct from `B-create-level-saved-true-no-umap` (tears down only create_level's own orphan, not a level.create phantom), `B-level-create-makes-wp-map` (what world create builds), `E-level-load-file-not-found-vs-in-memory-orphan` (load error wording; its `#2` confirms level.create still strands the orphan but files no discard verb), and `E-create-level-save-false-world-not-active` (activation verb, out of scope). Ripgrep over OPEN/closed for discard/unload/reset-world/set-active/evict found no ticket on the missing discard verb or the load-a-decoy eviction workaround.
