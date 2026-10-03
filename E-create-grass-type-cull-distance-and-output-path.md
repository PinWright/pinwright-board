---
id: E-create-grass-type-cull-distance-and-output-path
title: "landscape.create_grass_type takes no cull-distance parameters and leaves StartCullDistance == EndCullDistance == 10000 (no fade band; at that default the carpet stops inside a 160 x 240 m zone), and it never reaches disk without saying so — no save report in the response"
status: OPEN
severity: Medium
category: ergonomic
tags: [landscape, create_grass_type, grass, vegetation, cull-distance, fade-band, hardcoded-path, save-no-disk-write, performance-knob, parameter-gap]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
rice: [1, 2, 1, 1]
priority: 17
---

# landscape.create_grass_type: no cull-distance parameters, and an unwritten asset with no save report

`landscape.create_grass_type`
(`Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp:1614`) takes `name`, `meshPath`,
`savePath`, `density`, `minScale`, `maxScale` (`:1615-1626`) — no cull distance.

**1. Start == End, so grass pops instead of fading.** The variety block (`:1745-1764`) writes the
mesh, both density slots, scales, rotation and surface alignment, and never touches
`StartCullDistance` / `EndCullDistance`, so all four slots keep the engine constructor's 10000
(`FGrassVariety::FGrassVariety`, `Engine/Source/Runtime/Landscape/Private/LandscapeGrass.cpp:1486-1489`).
Equal start and end means a zero-width fade band: full-size instances up to the radius, then nothing.

**2. Measured: at 10000 the carpet stops inside the authored zone.** In a 160 x 240 m zone the far
half was bare. Proved to be the cull distance, not a weightmap edge, by moving only the camera: full
carpet at 3,905 uu, bare at 12,903 uu. Retuning to `12000 / 20000` fixed it at roughly 4x the
instance count, because the build radius is `EndCullDistance` and cost grows with its square. That
is why the ask is a parameter, not a bigger default.

**3. Not persisted, and the response does not say so.** `McpSafeAssetSave(GrassType)` (`:1766`) is
mark-dirty only (`Source/PinWright/Private/Utils/AssetUtils.cpp:553-567`), and the response
(`:1767-1834`) carries no `AddMarkDirtySaveReport` block (`AssetUtils.cpp:588`), so a caller cannot
tell the `.uasset` is not on disk. Part of the `*-save-no-disk-write` family
(`B-audio-create-save-no-disk-write` and siblings).

**Workaround:** after creation, set `GrassVarieties[0].StartCullDistance.Default` /
`EndCullDistance.Default` (and the `...Quality` slots) with `property.set`; persist with `asset.save`.

**Fix:**
1. Optional `startCullDistance` / `endCullDistance`, defaulting to 10000 so existing behaviour holds.
   Write both the per-platform and per-quality slot, as the handler already does for density
   (`:1755-1759`; `GetEndCullDistance()` reads the quality slot on hosts with
   `UseGrassVarityPerQualityLevels`).
2. Read `start_cull_distance` back off the stored variety beside the existing `end_cull_distance`
   (`:1779`).
3. Call `AddMarkDirtySaveReport` on the response (or persist the asset), so persistence state is
   measured, not implied.

**Acceptance:** `create_grass_type {startCullDistance: 12000, endCullDistance: 20000}` returns
`start_cull_distance: 12000` and `end_cull_distance: 20000`, and the stored variety reads those values
in both the `.Default` and `...Quality` slots; omitting them still yields 10000 / 10000; the response
carries the save-report fields stating the package is dirty and not yet on disk.

## History
- `#1-cull-distance-not-a-parameter` `OPEN` reporter — Filed off live measurement. At the default
  `EndCullDistance` 10000 the grass carpet stops inside a 160 x 240 m zone; proved to be the cull
  distance and not a weightmap edge by moving only the camera against a fixed weightmap — bare
  ground at 12,903 uu, full carpet at 3,905 uu. Retuning to 12000/20000 fixed it at roughly 4x the
  instance count, the build radius being `EndCullDistance` and cost growing with its square, which
  is the argument for a parameter over a larger default. Source re-derived: the handler registers
  five parameters, none a cull distance (`LandscapeHandler.cpp:1502`, params `:1503-1509`); the
  variety block (`:1587-1606`) never assigns `StartCullDistance` or `EndCullDistance`, so both keep
  the constructor's 10000 (`LandscapeGrass.cpp:1486-1489`) and start == end leaves no fade band;
  the output path is hardcoded at `:1556`; and `McpSafeAssetSave` at `:1608` is mark-dirty only
  (`AssetUtils.cpp:362-377`). Cross-linked `B-create-grass-type-addzeroed-never-renders` as the
  predecessor whose fix produced this starting state, and `B-asset-move-to-missing-folder-silently-renames`
  as the defect the hardcoded-path workaround hits.
- `#2-rephrased` `OPEN` developer — Re-checked at PinWright `7230b41d`. The hardcoded output path is fixed: `savePath` is now a parameter (`LandscapeHandler.cpp:1621-1623`) and the effective folder is echoed as `save_path` (`:1723`, `:1773`), so that ask, its section and the `B-asset-move-to-missing-folder-silently-renames` workaround note are dropped from the title and body. Still real and kept: no cull-distance parameters (variety block `:1745-1764` never writes them) and `McpSafeAssetSave` (`:1766`) with no `AddMarkDirtySaveReport` in the response. All line citations refreshed (verb moved from `:1502` to `:1614`; `AssetUtils.cpp` `McpSafeAssetSave` now `:553`). Severity unchanged (Medium).
