---
id: B-asset-dump-package-flags-always-empty
title: "asset.dump meta.json packageFlags array is empty for every dumped asset — flag detection never fires"
status: DONE
severity: Low
category: bug
tags: [asset-dump, meta, package-flags]
---

# asset.dump meta.json packageFlags array is empty for every dumped asset — flag detection never fires

`packageFlags: []` on every meta.json across a 30k-asset sweep. The flag table at `AssetDumpBuilder.cpp:72-81` maps 7 flags (`PKG_PlayInEditor`, `PKG_Cooked`, `PKG_ContainsMap`, `PKG_EditorOnly`, etc.).

Content packages on disk usually carry no flags at runtime, so the field carries zero signal in practice — could be dropped, or the table expanded to include flags that *would* fire (e.g. `PKG_FilterEditorOnly`, `PKG_NewlyCreated`, `PKG_Compiling`).

**Repro:**
1. Inspect any meta.json under `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/`.
2. Observe: `packageFlags: []` on every file across a 30k-asset sweep.

**Fix:** Drop the `packageFlags` field from `meta.json`. Bump dump schema version 4 → 5. Evidence: across ~30k dumped assets only 11 engine-shipped assets emit any flag (all `RuntimeGenerated`); `.umap`/`World` packages aren't dumped (dump-folder skips levels by default), so `PKG_ContainsMap` never fires; no in-plugin consumer reads the field — `TestAssetDumpBuilder` only asserted key presence. Wiki updated to drop the field from the documented meta.json shape.

## History
- `#1-initial-repro` `OPEN` reporter — `packageFlags: []` on every meta.json across a 30k-asset sweep. The current 7-flag table at `AssetDumpBuilder.cpp:72-81` only matches runtime states content packages don't normally carry. Field is degenerate.
- `#2-drop-degenerate-field` `IN-REVIEW` developer — Dropped `packageFlags` field from `meta.json` (`AssetDumpBuilder.cpp`). Bumped `dumpSchemaVersion` 4 → 5. Removed the field documentation from `docs/wiki/asset.md`. Flipped the test assertion in `TestAssetDumpBuilder.cpp` from key-present to key-absent and bumped its schema-version expectation. Evidence: 11/30k assets emitted only `RuntimeGenerated`; no map dumps to ever surface `PKG_ContainsMap`; no in-plugin consumer reads it.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/App/Track/Tunnel/tunnel_03` produced meta.json with no `packageFlags` key and `dumpSchemaVersion: 5` (was 3 in the prior cached dump). Fix matches IN-REVIEW claim.
