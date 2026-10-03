---
id: E-asset-dump-folder-cache-miss-reason-opaque
title: "asset.dump_folder reports dumped/unchanged counts but never why entries re-dumped — a mass re-dump after an engine, DumpCoreVersion or aspect-version change looks the same as a cache bug"
status: OPEN
severity: Low
category: ergonomic
tags: [asset-dump, dump-folder, cache, telemetry, fingerprint]
encounters: 2
costly: 1
lastSeen: 2026-09-09T07:55:00Z
rice: [1, 2, 1, 1]
priority: 17
---

# asset.dump_folder reports dumped/unchanged counts but never why entries re-dumped

When most of a folder re-dumps without `force`, the caller cannot tell "the cache is broken" from
"a version change invalidated it by design". On `/Game` that is the difference between an expected
22k-asset reload and a cache bug worth debugging before committing the editor to the sweep.

`AssetDumpCache::IsCacheFresh` (`Source/PinWright/Private/Handlers/Asset/AssetDumpCache.cpp:1172-1215`)
computes a per-asset reason (`"dumper fingerprint mismatch"`, `"source fingerprint mismatch"`,
`"package is dirty"`, `"cache version mismatch"`, `"options mismatch"`, ...). The dumper fingerprint
compares `DumpCoreVersion`, `EngineVersion` and the per-aspect versions
(`AreDumperFingerprintsEqual`, `:486-493`), so an engine update, an `AssetDumpCoreVersion` bump
(`AssetDumpCache.h:19`, now `4`) or an aspect-version bump in the `GetAspectVersion` table
(`AssetDumpCache.cpp:705-879`) invalidates every affected record at once. `PluginVersion` is no
longer compared (`:477-485`).

The folder path discards the reason. Both enumeration loops call the bool-only `IsDumpFresh`
(`AssetDumpHandler.cpp:1537` and `:3215`), which writes the reason only to the editor log
(`AssetDumpCache.cpp:1290-1292`) and returns a bool (`:1294`). The loops count `UnchangedCount` and
queue the rest. The started and completion payloads (`AssetDumpHandler.cpp:1285`, `:1412`, `:3658`)
carry `queued` / `dumped` / `unchanged` / `skipCount` and no reason.

**Workaround:** grep the editor log for `asset.dump cache: stale dump for` lines (one per queued
asset, with the reason), or diff a fresh `.dumpcache.json` `dumper` block against an old one.

**Fix:** make the folder enumeration use the reason-returning path (`IsCacheFresh`, or an `OutReason`
out-param on `IsDumpFresh`), count reasons in a `TMap<FString,int32>` at the miss branch, and emit
`staleReasonCounts` (for example `{"dumper fingerprint mismatch": 8161}`) in the started, progress
and completion payloads beside `unchanged` / `queued`. Not counted for `force: true`.

**Acceptance:** after an aspect-version bump, `asset.dump_folder` on a previously dumped folder
returns `staleReasonCounts` whose `"dumper fingerprint mismatch"` count equals `queued`. A sweep of an
unchanged folder returns an empty or absent `staleReasonCounts` with every asset `unchanged`.

Related: `B-dumpcache-fingerprint-includes-marketing-version` (removed `PluginVersion` from the
comparison), `B-asset-dump-folder-no-skip-event-or-counter` (DONE, the `skipCount` axis).

## History
- `#1-initial-finding` `OPEN` reporter — Routine `/App` re-sweep re-dumped 8161/8168 with 3 cache hits, no `force`. Root-caused as an expected dumper-fingerprint invalidation: last full sweep (92d87494fe, 07-10) ran at plugin VersionName 0.4.0; the 07-16 "refresh" (affa310718) re-dumped only 25 files; VersionName bumped 0.4.0→0.5.0→0.6.0 (01b1ffe5, 07-15), so ~8161 pre-bump markers failed `AreDumperFingerprintsEqual` on pluginVersion (`AssetDumpCache.cpp:478,768,1055`). NOT a bug. Filed as ergonomic because `asset.dump_folder`'s completion payload surfaces `dumped`/`unchanged` counts but never the invalidation reason — the per-asset `FAssetDumpFreshnessResult::Reason` is computed then discarded by the bool-only `IsDumpFresh` (`AssetDumpHandler.cpp:1796` → `AssetDumpCache.cpp:1123`), leaving an agent unable to tell "cache broken" from "version bump". Proposed fix: aggregate stale reasons into a `staleReasonCounts` field on the started/completion payload.
- `#2-second-version-bump-invalidation-on-game` `OPEN` reporter — Second encounter of the same mechanism, on a much larger tree; `encounters` 1 -> 2, `costly` deliberately left at 1. UE 5.8, `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`. The committed mirror was current in content, but the plugin `VersionName` had moved since it was written, so every `.dumpcache.json` failed `AreDumperFingerprintsEqual` on `PluginVersion` and the whole tree requeued — for `/Game` that is **22,446 assets**, which then OOM-killed the editor twice (`B-dump-folder-sweep-never-gcs-ooms-editor`). Source re-verified against this checkout: the fingerprint still compares `PluginVersion` at `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Asset\AssetDumpCache.cpp:481-486`, and `GetPluginVersion` at `:487-491` still reads the descriptor's `VersionName`. This ticket's ask is unchanged and still correct — the response should say *why* entries requeued — and the `staleReasonCounts` shape proposed in `#1` covers this case verbatim. What this encounter adds is that the reason is worth more than diagnosis time on a large tree: it is the difference between "expected, run it in subtrees" and "start debugging the cache" before committing an editor to a sweep that can crash it. `costly` not bumped: the diagnosis here cost one `.dumpcache.json` read and one source re-check, i.e. the cheap path, and the expensive part (the reload itself) is attributable to the fingerprint *policy* rather than to the missing telemetry. That policy is now disputed separately as `B-dumpcache-fingerprint-includes-marketing-version` (OPEN, bug) — filed rather than appended here because this ticket rules the invalidation correct-by-design and asks only for reporting; the two fixes are independent and both are wanted.
- `#3-rephrased` `OPEN` developer — The old title and body framed the mass re-dump as a plugin-version bump; `PluginVersion` is no longer compared (`AssetDumpCache.cpp:477-493`, `B-dumpcache-fingerprint-includes-marketing-version`), so the remaining triggers are engine version, `AssetDumpCoreVersion` (now 4, `AssetDumpCache.h:19`) and aspect-version bumps. Refreshed citations (`IsDumpFresh` callers now `AssetDumpHandler.cpp:1537`, `:3215`; `IsCacheFresh` `AssetDumpCache.cpp:1172-1215`), noted the per-asset log line as the workaround, and added Acceptance. The `staleReasonCounts` ask is unchanged and still unimplemented. Severity unchanged (Low).
