---
id: F-geometry-lod-screen-sizes-proper
title: "Reimplement geometry.set_lod_screen_sizes to write FStaticMeshSourceModel.ScreenSize (removed as a corrupting stub)"
status: OPEN
severity: Medium
category: feature
tags: [geometry, static-mesh, lod, screen-size, reimplement, rpc-cull]
---

# Reimplement `geometry.set_lod_screen_sizes` properly (per-LOD screen-size thresholds)

`geometry.set_lod_screen_sizes` was **removed** in the RPC cull recorded in
[`E-rpc-cull-151-record`](E-rpc-cull-151-record.md) because its implementation
was fake — worse than a no-op, it actively **corrupted** the mesh. The
capability it advertised (set the per-LOD screen-size transition thresholds on a
`UStaticMesh`) is legitimately wanted and has no clean generic replacement:
`property.set` cannot reach the per-LOD `FStaticMeshSourceModel` array entries,
and the sibling `geometry.set_lod_settings` owns `ReductionSettings`, not the
screen-size crossover values that decide when each LOD swaps in.

## What the removed version did wrong

The old handler (`Handlers/Geometry/LODCollisionHandler.cpp:444-445`) never wrote
any `ScreenSize` field. Instead it overwrote
`SourceModel.ReductionSettings.PercentTriangles` with the caller's screen-size
values, then ran `PostEditChange()` + save and reported `success:true`. So the
requested setting was never applied **and** it silently trashed the reduction
percentages that `geometry.set_lod_settings` manages — a corrupting write
disguised as success. That is why it was culled rather than left in place.

## Proper implementation

For each LOD index `i`:

- Write `StaticMesh->GetSourceModel(i).ScreenSize` to the requested value
  (`FPerPlatformFloat` — set the default value).
- Set `StaticMesh->bAutoComputeLODScreenSize = false` **once** so the manual
  screen sizes are honored (with auto-compute on, the engine recomputes and
  discards the caller's values).
- Do **not** touch `ReductionSettings.PercentTriangles` (that is
  `set_lod_settings`' contract).
- Then `PostEditChange()` + `McpSafeAssetSave`, matching the file's existing
  save pattern.

Validate the LOD index range and that the values are monotonically decreasing
(engine expectation for screen sizes) rather than accepting any array blindly.

**Fix:** New handler in `LODCollisionHandler.cpp` (the file kept its 4 other real
methods — `generate_complex_collision`, `simplify_collision`, `generate_lods`,
`set_lod_settings` — and the shared `FLodCollisionTarget`/`FindLodCollisionTarget`
helper, so reuse those). Add a regression test that authors a multi-LOD static
mesh, sets distinct screen sizes, and asserts `GetSourceModel(i).ScreenSize`
reflects them and that `bAutoComputeLODScreenSize == false` — and that
`ReductionSettings.PercentTriangles` is left untouched (the anti-corruption
guard the removed version failed).

## History
- `#1-reimpl-after-cull` `OPEN` reporter — Filed to reinstate the wanted capability removed by the RPC cull ([`E-rpc-cull-151-record`](E-rpc-cull-151-record.md)). The removed `geometry.set_lod_screen_sizes` was a corrupting stub: it wrote screen-size values into `ReductionSettings.PercentTriangles` (LODCollisionHandler.cpp:444-445) instead of `SourceModel.ScreenSize`, never set `bAutoComputeLODScreenSize=false`, and reported success — so it both failed to apply the setting and clobbered the reduction percentages owned by `set_lod_settings`. Proper impl: write `FStaticMeshSourceModel.ScreenSize` per LOD + set `bAutoComputeLODScreenSize=false`, leaving `ReductionSettings` alone. No generic workaround (`property.set` cannot reach per-LOD source-model fields).
