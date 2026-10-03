---
id: B-ortho-grass-block-last-tile-only
title: "render.capture_ortho_tiles publishes the grass block of the LAST tile only, so an earlier tile's culled-grass or unsettled grassWarning is dropped and only allTilesSettled survives"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_ortho_tiles, landscape, grass, grassWarning, aggregation, docs]
encounters: 1
lastSeen: 2026-10-03T00:00:00Z
rice: [1, 3, 1, 2]
priority: 17
---

# An ortho burst reports one tile's grass and calls it the burst's

`OrthoTileCaptureUtils.cpp:681` runs `SettleGrassForCapturePose` per tile and overwrites the
member `GrassBuild` each time (`:690` folds that tile's reach into it). The only cross-tile
state kept is `GrassBuildTotalMs` and `bAllTilesGrassSettled` (`:691-695`).
`OrthoTileCaptureHandler.cpp:797-806` then builds the response and manifest `grass` block from
`Capture.GetGrassBuild()` -- the last tile's report -- plus `buildMsTotal` and `allTilesSettled`.

Consequences:
- `grass.settled`, `grass.pendingComponents`, `grass.instances`, `grass.reach` and
  `grass.grassWarning` describe the last tile only. Per-tile entries in `tiles[]` carry no grass
  block.
- The culled-grass warning (`LandscapeGrassSettle.cpp:355-385`, built and settled but none of it
  in reach) is lost for every tile but the last, and there is no burst-level aggregate for it. An
  earlier tile that rendered bare ground because its grass was culled comes back with no signal.
- `allTilesSettled` is the one correct burst aggregate, and no wiki page documents it or
  `buildMsTotal` (`grep -rn allTilesSettled docs/wiki-src` is empty). `render.md`'s grass guidance
  tells callers to read `grass.settled` / `grass.pendingComponents` / `grassWarning`, which on this
  verb is a one-tile sample.

**Fix:** publish per-tile `grass` (or at least `settled`, `builtForPose`, `instancesInReach` and
any `grassWarning`) on each `tiles[]` entry, or add burst-level aggregates (`tilesUnsettled`,
`tilesGrassCulled` with tile coordinates) and a burst-level `grassWarning` when any tile warned.
Document the burst fields on `render.capture_ortho_tiles` in `docs/wiki-src/render.md`.

Severity: Medium. Misleading output on a less-used verb; `allTilesSettled` covers the unfinished
build case, so the silent gap is the per-tile culled case and the undocumented aggregates.

## History
- `#1-grass-block-is-last-tile-only` `OPEN` reviewer — Found while reviewing `B-capture-docs-prescribe-retired-grass-wait` `#2`, whose new `render.md` paragraph says every ortho tile builds its own grass and then tells callers to read `grass.settled` / `grass.pendingComponents` / `grassWarning`. Code at plugin `7230b41d`: `OrthoTileCaptureUtils.cpp:681-695` overwrites `GrassBuild` per tile; `OrthoTileCaptureHandler.cpp:797-806` serializes that last report plus `buildMsTotal` / `allTilesSettled`. Board grep for `allTilesSettled` / `last tile` returns only `B-ortho-capture-renders-no-landscape-grass` (IN-REVIEW), which added the per-tile build and the burst aggregate but does not cover the dropped per-tile warnings; filed separately rather than appended to a ticket under review.
