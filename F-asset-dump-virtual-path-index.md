---
id: F-asset-dump-virtual-path-index
title: "asset.dump_folder — emit a top-level index mapping UE virtual paths to dump dirs"
status: WONTFIX
severity: Medium
category: feature
tags: [asset-dump, navigation, virtual-path, index]
---

# asset.dump_folder — emit a top-level index mapping UE virtual paths to dump dirs

`properties.json` files reference other assets using raw UE virtual paths
(`/App/App/UI/W_FadeInOutBlack.W_FadeInOutBlack_C`,
`/App/HELIOS/Drones/Icarus/B_PioneerFPV.B_PioneerFPV_C`, etc.). To navigate
from such a ref to the corresponding dump folder on disk an agent has to
manually:

1. Strip the trailing `.SomeName_C` (class form) or `.SomeName` (object
   form) suffix to recover the package path.
2. Remap the virtual root: `/App/...` → `<asset-dumps-root>/App/App/...`,
   `/Game/...` → `<asset-dumps-root>/Game/...`, etc.

There is no top-level catalog of what the sweep produced, so an agent
cannot answer "does a dump exist for this ref?" without a directory probe
per ref. With ~30k dumped assets in this project, this churns context and
trial-reads.

## Concrete examples (current `properties.json` shape)

DataAsset → soft class pointer (TSoftClassPtr) — from
`App/App/DA_PioneerFPV/properties.json`:
```
"is_overridden_locally": true,
"type": "TSoftClassPtr<ADrone>",
"value": "/App/HELIOS/Drones/Icarus/B_PioneerFPV.B_PioneerFPV_C"
```

Blueprint (game-instance class) → hard class ref (TSubclassOf) — from
`App/App/LevelBlueprints/B_DroneGameInstance/properties.json`:
```
"is_overridden_locally": true,
"type": "TSubclassOf<UUserWidget>",
"value": "/App/App/UI/LobbyAndMenu/Popups/W_ForcedLogoutPopup.W_ForcedLogoutPopup_C"
```

Widget BP → activatable widget class ref — from
`App/App/UI/LobbyAndMenu/HUD/Training/Stabilized/W_HUD_TrainingStabilized_01/properties.json`:
```
"is_overridden_locally": true,
"type": "TSoftClassPtr<UCommonActivatableWidget>",
"value": "/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu.W_HUD_DroneGameMenu_C"
```

Non-class object ref (static mesh) — same `_C`-less dual-name shape, also
needs the same suffix strip — from
`App/PathTracer/Blueprints/BP_PathTracer/properties.json`:
```
"is_overridden_locally": true,
"type": "UStaticMesh*",
"value": "/App/PathTracer/Meshes/SM_Cylinder_Corner.SM_Cylinder_Corner"
```

In all four cases the dump dir is `<asset-dumps-root>/<root>/<package
path without dual suffix>`. The meta.json sidecar already records the
canonical package path (e.g.
`"assetPath": "/App/App/UI/W_FadeInOutBlack.W_FadeInOutBlack"`), so the
reverse mapping is recoverable per-asset — just not aggregated.

## Proposed shape

Emit a single `<asset-dumps-root>/index.json` once per
`asset.dump_folder` completion, written from `FinalizeAsyncDump` after
`ReconcileMirrorSubtree` runs (same place that already enumerates
`State.LiveDumpDirs`). Shape:

```json
{
  "schemaVersion": 1,
  "generatedAt": "2026-05-11T...Z",
  "sweptRoot": "/App",
  "entries": {
    "/App/App/UI/W_FadeInOutBlack": "App/App/UI/W_FadeInOutBlack",
    "/App/HELIOS/Drones/Icarus/B_PioneerFPV": "App/HELIOS/Drones/Icarus/B_PioneerFPV",
    ...
  }
}
```

Keys are the **package path only** (no dual-name suffix, no `_C`). A
consumer normalises any ref by stripping everything from the last `.`
onward (when the substring after the last `/` contains a `.`) and looks
up the result. One `O(1)` lookup replaces the manual probe.

Size estimate: ~30k entries × ~80 chars each (key + value) ≈ 2.4 MB.
Acceptable; if it grows past a few MB the file can be sharded by
top-level virtual root (`index-App.json`, `index-Game.json`, ...).

Scope notes:
- Emit only at the end of `asset.dump_folder` — single-asset `asset.dump`
  does not produce an index, since per-asset writes don't know the swept
  subtree. Single-asset writes that target a path inside a previously
  swept root leave the existing `index.json` stale until the next sweep
  (acceptable; documented).
- Index is **scoped to the swept subtree**, same as reconciliation —
  re-running `asset.dump_folder /App` overwrites entries under `/App/`
  only and leaves entries under `/Game/` intact. Easiest implementation:
  load existing `index.json`, drop entries whose key is a child of
  `sweptRoot`, merge the live set, write back.
- Symlinks for suffix-less / `_C`-form aliases are explicitly **not**
  proposed — Windows symlinks need admin or developer mode and would
  double the on-disk inode count.

**Workaround:** Per-asset `meta.json` already contains `assetPath` in
the dual-name form; a consumer can glob every `meta.json` and build the
index client-side. Slow and wastes context but works.

**Fix:** New helper in `AssetDumpHandler.cpp` near `FinalizeAsyncDump`
that takes `State.LiveDumpDirs` + `State.RootDir`, derives package-path
keys from each dir's path-relative-to-root, merges with the existing
`index.json` (scoped to `sweptRoot`), and writes atomically (`.tmp` +
rename) using `AssetDumpWriter`'s existing helpers. No new RPC method
needed.

## History
- `#1-initial-feature-request` `OPEN` reporter — Cross-references in
  `properties.json` use UE virtual paths and consumers must hand-roll the
  ref→dir mapping (strip `.<Name>_C`/`.<Name>` suffix, prepend dump root,
  preserve project-root segment). With ~30k dumped assets the missing
  catalog forces repeated directory probes. Proposed minimal addition:
  emit `<asset-dumps-root>/index.json` from `FinalizeAsyncDump` after
  reconciliation, mapping package path → dump dir relative path, scoped
  to the swept subtree (entries outside `sweptRoot` are preserved). Size
  ~2-3 MB for this project; shardable by top-level virtual root if it
  grows. No new RPC, no change to per-asset files.
- `#2-wontfix` `WONTFIX` user — Not worth implementing.
