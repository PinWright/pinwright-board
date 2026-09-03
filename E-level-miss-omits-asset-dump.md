---
id: E-level-miss-omits-asset-dump
title: "LEVEL_NOT_LOADED and the level wiki offer only level.load — never asset.dump, which reads an unopened .umap in full without touching the active world"
status: OPEN
severity: Medium
category: ergonomic
tags: [level, asset-dump, level-not-loaded, signpost, discoverability, wiki, active-world]
encounters: 1
lastSeen: 2026-08-18T00:00:00Z
---

# The one remedy the miss-message offers is the one that mutates the active world

`asset.dump` reads an **unopened** `.umap` in full — `world_settings.json`, `level_bp.txt`,
`sublevels.json`, `actors/manifest.json` and one JSON per actor — through `LoadObject`, without
swapping the active world (`Handlers/Asset/AssetDumpHandler.cpp:758, 2084`). The capability is
fine. The signpost is missing.

`SendLevelInspectMissError` (`Handlers/Level/LevelLoadDiagnostics.h`) told a caller who asked
`level.get_info` / `get_actors` / `get_bounds` about an on-disk-but-unloaded map:

> Level '%s' exists on disk but is not loaded into the active world; these getters read only
> loaded levels. **level.load it first**, or pass no levelPath to read the active level.

`level.load` makes the named map the active world. For a caller inspecting a map *other* than the
one they are working in, that is the single thing they cannot afford — and it is the only option
the message gave. `Docs/wiki-src/level.md:15` repeated the same one-option remedy verbatim.

The sentence to emit already existed and was already wired up one path over:
`AssetDumpSuggestion::BuildDumpSuggestionHint` is called on `level.get_actors`' **success** path
but not on the error path, which is where a stuck caller actually lands.

## Measured cost

This session two agents needed the actor inventory of `L_FootballArena.umap` and
`L_FootballBootstrap.umap`. Neither map was the active world — an editor was open and serving the
user's own flight testing, so `level.load` was not an acceptable remedy, which is precisely the
case the message does not address. With no other option named, both fell back to **binary
string-scanning the `.umap` files** — twice — for structured information `asset.dump` returns
directly.

That is the ergonomic shape this board rates: the task was doable, but only through a fallback
invented on the spot, because the error message pointed at the one remedy the caller had ruled
out and never mentioned the one that fit.

## Distinct from `E-level-getters-require-loaded-not-on-disk`

That ticket (IN-REVIEW) is the reason `LEVEL_NOT_LOADED` exists at all: it replaced a misleading
`LEVEL_NOT_FOUND` for a map that is demonstrably on disk. Its Fix explicitly scoped out
non-mutating alternatives — *"the over-scoped alternative — auto-loading / peeking an unloaded
on-disk map for a read getter ... is intentionally out of scope: it drags active-world
mutation/restore risk into a read-only path"*. Correct about auto-loading, wrong about the
conclusion drawn from it: a non-mutating read of an unloaded map already ships as `asset.dump`, so
the honest message has two remedies, not one. This ticket is about the **content of the message**,
not the verdict it carries.

**Fix:** name `asset.dump` first in the `LEVEL_NOT_LOADED` message, stating that it reads a map
without opening it, and call `BuildDumpSuggestionHint` on the error path the way the success path
already does. One clause in `Docs/wiki-src/level.md:15` to match.

**Status note:** the remedy was authored this session by a sibling agent and currently sits as an
**uncommitted working-tree change** in the plugin clone — `Docs/wiki-src/level.md`,
`Source/PinWright/Private/Handlers/Level/LevelLoadDiagnostics.h` (now `#include`s
`Utils/AssetDumpSuggestion.h`, names `asset.dump` first, appends `BuildDumpSuggestionHint`), and
`Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp`. Left `OPEN` because that change is
neither committed nor built nor tested; a developer flips it to `IN-REVIEW` once it lands.

## Related

- `E-level-getters-require-loaded-not-on-disk` (IN-REVIEW) — created `LEVEL_NOT_LOADED`; this is
  the missing second remedy in the message it introduced.
- `F-asset-dump-folder-skip-levels-by-default`, `F-dump-world-metadata` — the level-dump surface
  this signpost points at.

## History
- `#1-signpost-missing-on-the-error-path` `OPEN` reporter — `asset.dump` reads an unopened `.umap` in full (`world_settings.json`, `level_bp.txt`, `sublevels.json`, `actors/manifest.json`, one JSON per actor) via `LoadObject` without swapping the active world (`AssetDumpHandler.cpp:758, 2084`), but nothing points a stuck caller at it: `SendLevelInspectMissError` (`Handlers/Level/LevelLoadDiagnostics.h`) offered only "`level.load` it first, or pass no levelPath", and `Docs/wiki-src/level.md:15` repeated it — `level.load` being the one remedy that mutates the active world, which is exactly what a caller inspecting a different map cannot do. `AssetDumpSuggestion::BuildDumpSuggestionHint` already generates the right sentence and was already wired into `level.get_actors`' success path, but not the error path where callers land. Measured cost this session: two agents needing the actor inventory of `L_FootballArena.umap` and `L_FootballBootstrap.umap` — neither the active world, with an editor open for the user's flight testing so `level.load` was ruled out — fell back to **binary string-scanning the `.umap` files, twice**, for information `asset.dump` returns as structured JSON. Distinct from `E-level-getters-require-loaded-not-on-disk` (IN-REVIEW), which created `LEVEL_NOT_LOADED` and deliberately scoped out auto-load/peek as active-world mutation risk: correct about auto-loading, but the conclusion that `level.load` is the remedy is wrong when a non-mutating read already ships. This ticket is the message content, not the verdict. Remedy authored this session by a sibling agent under the arena-rescaling plan's `wave-1-pinwright-level-signpost` chunk and currently uncommitted in the plugin working tree (`Docs/wiki-src/level.md`, `LevelLoadDiagnostics.h`, `Tests/World/TestLevelHandlers.cpp`); left OPEN until it is committed, built and tested.
