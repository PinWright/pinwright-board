---
id: E-level-getters-require-loaded-not-on-disk
title: "level.get_info / get_actors / get_bounds resolve a levelPath only for an already-loaded level; an on-disk-but-unloaded map returns misleading [LEVEL_NOT_FOUND] instead of an actionable LEVEL_NOT_LOADED"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [level, get_info, get_actors, get_bounds, levelPath, loaded-only, error-message, level-not-found, level-not-loaded, wiki]
---

# `level.get_info` (and `get_actors` / `get_bounds`) only inspect a *loaded* level — a `levelPath` for an unloaded-but-on-disk map errors `LEVEL_NOT_FOUND`

`level.get_info`, `level.get_actors`, and `level.get_bounds` all resolve their
`levelPath` argument through `FindLevelByPathLevel(World, LevelPath)`
(`LevelHandler.cpp:74-88`). That helper only walks
`GetAllLevelsFromWorldLevel(World)` — the **persistent level plus the sublevels
currently attached to the active editor world**. It never consults disk or the
asset registry. So when `levelPath` names a real map that exists on disk but is
not currently loaded into the active world, the helper returns `nullptr` and the
handler emits:

```
[LEVEL_NOT_FOUND] Level not found: /Game/Maps/Lighting/Lighting_Realtime
```

This message is misleading: the level is **not** "not found" — it exists on
disk, `level.list` lists it, and `level.load` opens it fine. The real condition
is "that level is not currently loaded into the active world, and these getters
only read loaded levels." The param docs said only *"Defaults to the active
editor level"* (`get_info` `:1063`, `get_actors` `:1385`, `get_bounds` `:1431`) and
gave no hint that a **non-active** path must already be loaded — so a caller
reasonably passes the package path of any map they want to inspect, and gets a
NOT_FOUND that reads like a path-resolution failure.

## Process friction (this task)

The audited task was a duplicate-and-light round-trip whose plan required reading
the **source** map's baseline actor count (step 2) and re-confirming the source
was untouched at the end (step 9) — for a map that was never the active world.
Both reads forced an otherwise-unnecessary `level.load` detour:

- `level.get_info {levelPath:/Game/Maps/Lighting/Lighting_Realtime}` (not loaded)
  → `[LEVEL_NOT_FOUND] Level not found: /Game/Maps/Lighting/Lighting_Realtime`
  → `level.load` the source → `level.get_info` (now succeeds, 54 actors).
- Later, after the active world had moved to the duplicate:
  `level.get_info {levelPath:/Game/Maps/Lighting/Lighting_Realtime}` (not loaded
  again) → the same `[LEVEL_NOT_FOUND]` → `level.load` the source again (just to
  verify it still reads 54) → `level.get_info`.

Friction note verbatim: *"level.get_info/get_actors/get_bounds only resolve
LOADED levels (path arg returns LEVEL_NOT_FOUND for an on-disk-but-unloaded map),
so I had to load the source to capture its baseline."* Each "read a map's info"
intent cost an extra `level.load` (and, on the verify step, mutated the active
world the caller had carefully set to the duplicate — re-loading the source to
read it un-does the very state the task was holding).

## Why this is ergonomic (process), not the duplicate bug

Distinct from the judge-filed `E-level-duplicate-in-memory-only-no-persist-signal`
(the duplicate's in-memory-only persistence signal) and from the IN-REVIEW
`E-level-load-file-not-found-vs-in-memory-orphan` (which split FILE_NOT_FOUND vs a
registered-but-unsaved world for the *load* path). This ticket is about the
**inspection getters** giving up on a perfectly-on-disk map and forcing a load to
read it. It is the same disk-vs-loaded blind spot the load fix addressed, but on a
different surface (`get_info`/`get_actors`/`get_bounds`) and with the opposite
failure mode: here the level genuinely *is* persisted on disk, yet the getter
still can't read it without a prior load.

**Fix:** Loaded-only inspection is the intended contract — so say so, and stop
calling it NOT_FOUND. Replace the misleading `LEVEL_NOT_FOUND "Level not found"`
for an on-disk-present map with a distinct, actionable verdict `LEVEL_NOT_LOADED`:
"Level '<path>' exists on disk but is not loaded into the active world; these
getters read only loaded levels — `level.load` it first, or pass no `levelPath`
to read the active level." Reserve `LEVEL_NOT_FOUND` for a path with **no** `.umap`
on disk. `LEVEL_NOT_LOADED` is already an established in-plugin code
(`ErrorCodes.h` `ERR_LEVEL_NOT_LOADED`; emitted by `level.set_world_settings`
`:1128` and `level.set_lighting` `:1171` for the same "requested level isn't
loaded" condition), so this follows existing prior art rather than inventing a
verdict. Disk existence is probed the same way `level.load` probes it
(`FFileManager::FileExists` on the converted `.umap` path + `FPackageName::DoesPackageExist`,
`LevelHandler.cpp:175-182`). Also update the `levelPath` param docs on all three
methods to state that a non-active path must already be loaded.

The over-scoped alternative — auto-loading / peeking an unloaded on-disk map for a
read getter (load-then-restore or a registry peek to report `actorCount` / actors
/ bounds without making it active) — is intentionally **out of scope**: it drags
active-world mutation/restore risk into a read-only path for a marginal payoff
(the verify-step re-load that the audit ran already corrupted the held active
world). The honest-verdict rename is the consistent, low-risk fix.

## Workaround

Before `level.get_info` / `get_actors` / `get_bounds` on a map you have not
loaded, `level.load` it first (then re-load whatever world you actually wanted
active). This is the detour the audited task ran twice.

## Docs/wiki overlay to improve

`docs/wiki-src/level.md` — the `level.get_info`, `level.get_actors`, and
`level.get_bounds` sections should state that the `levelPath` argument inspects
only levels **currently loaded into the active world** (the active persistent
level + loaded sublevels), that an on-disk-but-unloaded map returns
`LEVEL_NOT_FOUND` despite existing, and that to read an unloaded map you must
`level.load` it first. (The handler change above is the durable fix; the wiki
edit is the discoverability stopgap and is a downstream wiki process, not part of
this audit.)

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `level.duplicate`
  duplicate-and-light round-trip. The plan needed the **source** map's baseline
  actor count (step 2) and a final "source untouched" re-check (step 9) for a map
  that was never the active world; both forced a `level.load` detour because
  `level.get_info {levelPath:/Game/Maps/Lighting/Lighting_Realtime}` returned
  `[LEVEL_NOT_FOUND] Level not found: /Game/Maps/Lighting/Lighting_Realtime`
  twice (once before any load, once after the active world had moved to the
  duplicate) — even though `level.list` listed that exact map both times. Root
  cause: `level.get_info` (`LevelHandler.cpp:1033`), `level.get_actors`
  (`:1355`), and `level.get_bounds` (`:1401`) all resolve `levelPath` via
  `FindLevelByPathLevel` (`:73-87`), which walks only
  `GetAllLevelsFromWorldLevel(World)` (active world's loaded levels) and never
  checks disk/registry, so an on-disk-but-unloaded map misses and emits a
  NOT_FOUND that contradicts the map's real on-disk existence. (Source line
  citations refreshed in `#2-reword`: helper `:74-88`; get_info `:1081`,
  get_actors `:1403`, get_bounds `:1449`.) Friction note
  verbatim: "level.get_info/get_actors/get_bounds only resolve LOADED levels
  (path arg returns LEVEL_NOT_FOUND for an on-disk-but-unloaded map), so I had to
  load the source to capture its baseline." Distinct PROCESS angle from the
  judge-filed `E-level-duplicate-in-memory-only-no-persist-signal` (duplicate
  persistence signal) and the IN-REVIEW `E-level-load-file-not-found-vs-in-memory-orphan`
  (load-path FILE_NOT_FOUND vs in-memory orphan): this is the read-only
  *inspection* getters refusing an on-disk map and forcing a load to read it.
  Dedup: ripgrep across OPEN + closed found no ticket on the
  get_info/get_actors/get_bounds loaded-only resolution or the
  on-disk-map-NOT_FOUND wording (no board file contains `LEVEL_NOT_FOUND`;
  `B-level-get-bounds-ignores-actors` is the degenerate-zero-box bug,
  orthogonal).
- `#2-reword` `OPEN` developer — Validity pass (correctness + adversarial +
  board-historian). Defect confirmed present in current source and reproducible by
  source reasoning. Refreshed the stale line citations (helper `:74-88`; get_info
  `:1081`, get_actors `:1403`, get_bounds `:1449`; param docs `:1063` / `:1385` /
  `:1431`) and narrowed the fix to the in-pattern honest-verdict rename
  (`LEVEL_NOT_LOADED` for an on-disk-but-unloaded map + param-doc updates),
  dropping the over-scoped auto-load/peek alternative the adversarial lens flagged
  as dragging active-world mutation risk into a read-only getter. Not a duplicate
  (the two IN-REVIEW siblings cover the load path and the duplicate path, not the
  read-only getters; `B-level-get-bounds-ignores-actors` is the degenerate-box
  bug). Not already-fixed (the three getters still emit `LEVEL_NOT_FOUND`).
- `#3-fix` `IN-REVIEW` developer — Implemented the narrowed fix. In
  `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp`: added
  `DoesMapPackageExistOnDisk` (mirrors `level.load`'s `FFileManager` +
  `FPackageName::DoesPackageExist` probe) and `SendLevelInspectMissError`, then
  routed all three inspection getters (`level.get_info`, `level.get_actors`,
  `level.get_bounds`) through it: a non-empty `levelPath` that misses the loaded
  levels but has a `.umap` on disk now emits `LEVEL_NOT_LOADED` ("…exists on disk
  but is not loaded into the active world; … level.load it first, or pass no
  levelPath…"); a path with no map on disk keeps `LEVEL_NOT_FOUND`. The three
  mutation handlers that also use `FindLevelByPathLevel` (remove_from_world,
  set_visibility, set_locked) are deliberately untouched — out of ticket scope.
  Updated the `levelPath` param docs on all three getters to state the path must
  already be loaded and name the `LEVEL_NOT_LOADED` verdict. Wiki overlay
  `docs/wiki-src/level.md` updated (the `level.get_*` gotcha now describes the
  loaded-only resolution + `LEVEL_NOT_LOADED`). Regression test: three new
  automation tests in
  `Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp`
  (`PinWright.level.get_info|get_actors|get_bounds.UnloadedOnDiskMapReportsNotLoaded`)
  drive each production handler with a real on-disk map that is not the active
  world and assert the verdict is `LEVEL_NOT_LOADED` (not `LEVEL_NOT_FOUND`), the
  message names `level.load`, and the `levelPath` param doc warns loaded-only —
  all fail if the fix is reverted. Skips gracefully without an editor / alternate
  on-disk map. Not compiled here (a later phase compiles + runs).
