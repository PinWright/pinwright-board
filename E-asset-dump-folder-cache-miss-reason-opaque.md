---
id: E-asset-dump-folder-cache-miss-reason-opaque
title: "asset.dump_folder reports dumped/unchanged counts but never why entries re-dumped — a routine plugin-version bump invalidates the whole cache indistinguishably from a bug"
status: OPEN
severity: Low
category: ergonomic
tags: [asset-dump, dump-folder, cache, telemetry, fingerprint]
encounters: 1
costly: 1
lastSeen: 2026-07-17T07:06:00Z
---

# asset.dump_folder reports dumped/unchanged counts but never why entries re-dumped

A routine `asset.dump_folder {folderPath: "/App"}` re-dumped **8161/8168** assets
with only **3** cache hits: `{"assetCount":8168,"queued":8165,"dumped":8161,"unchanged":3,"skipCount":4}`.
The sweep cost ~12 min of editor churn and rewrote thousands of byte-identical
`meta.json`/`properties.json` files (mass CRLF-warning churn in git). `force` was
**not** passed, so the expectation was a near-total cache hit on a tree whose
committed mirror looked recent.

## This specific case is NOT a cache bug — it is a version-bump invalidation, by design

The cache freshness check (`AssetDumpCache::IsCacheFresh`, `AssetDumpCache.cpp:1018-1075`)
invalidates an entry when its stored **dumper fingerprint** differs from the current
one. The dumper fingerprint (`MakeCurrentDumperFingerprint`, `AssetDumpCache.cpp:763-771`;
compared field-by-field in `AreDumperFingerprintsEqual`, `:474-482`) folds in
`DumpCoreVersion`, `EngineVersion`, **`PluginVersion`**, and per-aspect versions.
`PluginVersion` is the plugin descriptor's `VersionName` read live from the running
editor (`GetPluginVersion`, `:484-488`).

Verified cause of the 8161 misses:
- The last **full** `/App` sweep was outer-repo commit `92d87494fe` (2026-07-10),
  when the plugin `VersionName` was **`0.4.0`** (`git show 8504ef5d:PinWright.uplugin`).
- The 2026-07-16 refresh (`affa310718`) that made the mirror *look* recent touched
  only **25 files (~7 Sumo assets)** — a targeted re-dump, not a full sweep. The
  bulk of the ~8161 `.dumpcache.json` markers still dated from the 07-10 full sweep.
- `VersionName` then bumped `0.4.0 → 0.5.0` and `0.5.0 → 0.6.0` (`01b1ffe5`, 07-15).
  Today's editor runs `0.6.0` (fresh `.dumpcache.json` now shows
  `"pluginVersion": "0.6.0"`, `"engineVersion": "5.7.4-51494982+++UE5+Release-5.7"`,
  `"dumpCoreVersion": 1`).
- So every pre-07-15 marker failed `AreDumperFingerprintsEqual` on `PluginVersion`
  → `Stale("dumper fingerprint mismatch")` → re-dump. `DumpCoreVersion` (still `1`
  since 06-11) and the `meta.json`/`properties.json` aspect versions (5/6, last
  bumped 07-10) were stable across the window and are not the cause.
- On-disk confirmation: ~8161 `.dumpcache.json` files carry a 2026-07-17 09:54–10:06
  (+0300) mtime (today's re-dump); only ~7 predate today (the 3 unchanged + 4 skipped).
  The cache was warm, not cold/machine-migrated — it was invalidated wholesale.

## The actual ergonomic gap

The `dumped`/`unchanged`/`skipCount` completion payload never says **why** entries
requeued. `IsCacheFresh` computes a precise per-asset reason string
(`FAssetDumpFreshnessResult::Reason`, e.g. `"dumper fingerprint mismatch"`,
`"source fingerprint mismatch"`, `"package is dirty"`), but the folder-dump caller
uses the **bool-only** `IsDumpFresh` overload (`AssetDumpHandler.cpp:1796`), which
discards `.Reason` at the boundary (`AssetDumpCache.cpp:1123`). The enumeration loop
(`AssetDumpHandler.cpp:1793-1810`) only increments `UnchangedCount`; a miss just falls
through to `Pending.Add`.

Consequence: an agent (or the user) seeing 8161/8168 re-dumps **cannot distinguish**
"the cache is broken / a bug requeued everything" from "a plugin/engine version bump
legitimately invalidated the whole cache." Diagnosing this one required reading
`AssetDumpCache.cpp` plus git archaeology across two repos. A single aggregated
reason line in the response would have made it self-explanatory ("re-dumped 8161:
dumper fingerprint mismatch — pluginVersion 0.4.0→0.6.0").

Distinct from `B-asset-dump-folder-no-skip-event-or-counter` (DONE) and
`B-asset-dump-game-high-skip-ratio` (WONTFIX): both are about **`skipCount`**
(assets that failed to load / were filtered). This is about the **`dumped` vs
`unchanged`** cache split — a different axis with no reason surfaced.

**Workaround:** none needed for correctness (output is correct). To confirm a mass
re-dump is expected rather than a bug, diff the fresh `.dumpcache.json`
`dumper.pluginVersion`/`engineVersion` against the plugin descriptor + git history.

**Fix (proposed):** Thread the stale reason through the folder-dump enumeration.
Swap the `IsDumpFresh` call at `AssetDumpHandler.cpp:1796` for the reason-returning
`IsCacheFresh` overload (or add a `FString& OutReason` out-param to `IsDumpFresh`),
accumulate a `TMap<FString,int32> StaleReasonCounts` at the miss branch
(`:1809`), and emit it as `staleReasonCounts: { "dumper fingerprint mismatch": 8161, ... }`
in the started/completion payloads alongside `unchanged`/`queued`. Cheap
(one map increment per asset) and makes version-bump invalidation self-evident.

## History
- `#1-initial-finding` `OPEN` reporter — Routine `/App` re-sweep re-dumped 8161/8168 with 3 cache hits, no `force`. Root-caused as an expected dumper-fingerprint invalidation: last full sweep (92d87494fe, 07-10) ran at plugin VersionName 0.4.0; the 07-16 "refresh" (affa310718) re-dumped only 25 files; VersionName bumped 0.4.0→0.5.0→0.6.0 (01b1ffe5, 07-15), so ~8161 pre-bump markers failed `AreDumperFingerprintsEqual` on pluginVersion (`AssetDumpCache.cpp:478,768,1055`). NOT a bug. Filed as ergonomic because `asset.dump_folder`'s completion payload surfaces `dumped`/`unchanged` counts but never the invalidation reason — the per-asset `FAssetDumpFreshnessResult::Reason` is computed then discarded by the bool-only `IsDumpFresh` (`AssetDumpHandler.cpp:1796` → `AssetDumpCache.cpp:1123`), leaving an agent unable to tell "cache broken" from "version bump". Proposed fix: aggregate stale reasons into a `staleReasonCounts` field on the started/completion payload.
