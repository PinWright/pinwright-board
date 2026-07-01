---
id: B-asset-dump-game-high-skip-ratio
title: "asset.dump_folder /Game has ~18× skip ratio vs /App — all ASSET_LOAD_FAILED, concentrated in third-party UltraDynamicSky duplicates"
status: WONTFIX
severity: Low
category: ergonomic
tags: [asset-dump, skips, third-party, telemetry]
---

# asset.dump_folder /Game has ~18× skip ratio vs /App — all ASSET_LOAD_FAILED, concentrated in third-party UltraDynamicSky duplicates

## Observed

Latest sweep:

| Subtree | Total | Dumped | Skipped | Skip % |
|---------|-------|--------|---------|--------|
| `/App`  | 8073  | 8069   | 4       | 0.05%  |
| `/Game` | 22361 | 22154  | 207     | 0.92%  |

`/Game` skip rate is ~18× higher than `/App`. Investigation against the on-disk skip stubs (`.editor-automation/asset-dumps/{App,Game}/**/meta.json` with `"skipped": true`) shows the skew is **not** a /Game-vs-/App-load-path bug — it is third-party content pack residue.

## Skip category breakdown (from skip-stub meta files)

**`/Game` — 207 skips, all `ASSET_LOAD_FAILED`:**

| Top-level | Count |
|-----------|-------|
| `UltraDynamicSky/Materials`   | 89  |
| `UltraDynamicSky/Blueprints`  | 67  |
| `UltraDynamicSky/Textures`    | 32  |
| `UltraDynamicSky/Meshes`      | 8   |
| `ChemicalPlantEnv/Meshes`     | 8   |
| `UltraDynamicSky/Particles`   | 1   |
| `SportsStadium/Meshes`        | 1   |
| `Building/Lighting`           | 1   |

Total UltraDynamicSky: **197/207 (95.2%)**.

**`/App` — 4 skips, all `ASSET_LOAD_FAILED`:**
- `/App/Gates/Models/M_gate_02_gate_02_B_geo_gate_syntethic_<HASH>`
- `/App/Meadow_Environment_Set2/Environment/Rocks/Meshes/M_SM_Cliff_overpaint_02_MI_Cliff_Leaves_Cover_<HASH>`
- `/App/Meadow_Environment_Set2/Environment/Rocks/Meshes/M_SM_Cliff_piece_01_MI_Cliff_Leaves_Cover_<HASH>`
- `/App/Meadow_Environment_Set2/Environment/Rocks/Meshes/M_SM_Cliff_piece_02_MI_Cliff_Leaves_Cover_<HASH>`

The four /App misses look like the same pattern but from `Meadow_Environment_Set2` (a third-party Meadow pack vendored into /App), not project-authored code.

Only `ASSET_LOAD_FAILED` is observed in the data; the other documented skip codes (`PATH_TOO_LONG`, `DUMP_WRITE_FAILED`, `NOT_SUPPORTED`) did not fire on either subtree this sweep. `ASSET_NO_BASELINE` is a diff-mode-only marker, not really a skip.

## Root cause: orphan `*_2.uasset` files in UltraDynamicSky

`find Content/UltraDynamicSky -name "*_2.uasset" | wc -l` returns **207** — exact match with the /Game skip count. Every UDS skip is a `<AssetName>_2.uasset` file that the AssetRegistry enumerates but `LoadObject` returns null for. Example: `UDS_CachedProperties_2.uasset` (43 KB, dated 2025-02-27) sits next to a fully-loadable `UDS_CachedProperties.uasset` (43 KB, dated 2025-01-16). Both files exist on disk with similar sizes; the `_2` copy is unreadable. Almost certainly a botched UDS update or marketplace-asset re-import that left two copies side by side.

**This is not a plugin bug.** The dump pipeline correctly enumerates these on-disk packages, attempts to load, fails, and writes a skip-stub. It's third-party content hygiene.

## What's actually missing

While investigating, two ergonomic gaps surfaced in the dump telemetry that make this kind of skew hard to triage without manually grepping the dump cache:

1. **No per-reason aggregation in the completion result.**
   `FinalizeAsyncDump()` (`Handlers/Asset/AssetDumpHandler.cpp:847-848`) emits only a flat `dumped` + `skipCount`. To learn that all 207 skips are `ASSET_LOAD_FAILED` concentrated in UDS, an operator must `grep -rh '"skipReason"' .editor-automation/asset-dumps/Game/ | sort | uniq -c`. The completion payload could carry a `skipCountsByReason: { ASSET_LOAD_FAILED: 207, ... }` object derived during the ticker walk (a single `TMap<FString,int32>` increment per skip).
2. **No `jobs.jsonl` event per skip.** Confirmed by reading `Utils/JobMonitorLog.h` (path constant) and `Handlers/Asset/AssetDumpHandler.cpp:1006-1007` — only `dumped N / M` progress lines are appended. The authoritative skip record is the on-disk stub `meta.json`. That's fine for offline inspection, but a live consumer watching `jobs.jsonl` via `Monitor` cannot see "asset X failed" in stream order. Per `B-asset-dump-folder-no-skip-event-or-counter.md` history note `#2`, this was an intentional design decision ("on-disk stub is the authoritative skip record"). Reconsider if streaming consumers become a use case.

## Recommendations

**Not a fix request** — this is logged as a finding so the next operator looking at skip-ratio drift has prior art.

If telemetry is touched anyway:
- Add `skipCountsByReason` (`TMap<FString,int32>`) to `FAsyncFolderDumpState`, increment in the skip branch at `AssetDumpHandler.cpp:923`, emit as JSON object in `FinalizeAsyncDump()` alongside `skipCount`.
- Optional: include a top-N list of skipped paths per reason (truncated) so the completion event is self-contained.

The 207 UDS skips themselves are a **content-side** problem — either delete the `*_2.uasset` orphans, or open a ticket against the UDS pack vendor. Outside this plugin's scope.

## History
- `#1-initial-finding` `OPEN` reporter — /Game skip rate (0.92%) is ~18× /App's (0.05%); broke down skip-stub `meta.json` files on disk: all 207 /Game + all 4 /App skips are `ASSET_LOAD_FAILED`, with 197/207 in UltraDynamicSky third-party content (orphan `*_2.uasset` copies that the AssetRegistry enumerates but LoadObject rejects). Not a plugin bug — third-party content hygiene. Two ergonomic gaps surfaced: completion result has no per-reason aggregation (forces operator to grep dump stubs); no per-skip event in `jobs.jsonl` (intentional per `B-asset-dump-folder-no-skip-event-or-counter.md` #2). Recommend (optional) `skipCountsByReason` on the completion payload if telemetry is touched.
- `#2-wontfix-third-party-content` `WONTFIX` developer — The 18× skew is content-side (orphan `*_2.uasset` files in vendored UDS / Meadow packs), not a plugin defect. The plugin correctly enumerates, attempts load, fails, writes skip-stub. The two surfaced telemetry gaps (per-reason aggregation in completion payload; per-skip `jobs.jsonl` event) are tracked elsewhere — the per-skip event has already been deliberately declined in `B-asset-dump-folder-no-skip-event-or-counter` #2 (on-disk stub is the authoritative record), and per-reason aggregation is a tiny QOL-only change that doesn't justify its own ticket. Closing as WONTFIX; if telemetry is ever touched for another reason, add `skipCountsByReason` opportunistically.
