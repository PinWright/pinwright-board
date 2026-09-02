---
id: B-dump-reconcile-blind-to-metaless-dirs
title: "Mirror reconcile is keyed on meta.json, so a dump dir that keeps only its gitignored .dumpcache.json is unreachable by both prune passes and leaves a current-fingerprint cache record for files that are not on disk"
status: OPEN
severity: Medium
category: bug
tags: [asset-dump, dump-folder, reconcile, prune, dumpcache, cache, mirror]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# Mirror reconcile is blind to dump dirs that lost their `meta.json`

`ReconcileMirrorSubtree` (`Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp:2274`)
finds prune candidates with a recursive `FindFilesRecursive` for `meta.json`
(`:2285-2288`) and tree-deletes each dir not in `LiveDirs`. A dump dir with **no**
`meta.json` is therefore never a candidate. The post-order second pass added by
`E-asset-dump-prune-empty-dirs` (`:2308-2324`) cannot finish the job either: it
calls `DeleteDirectory(..., Tree=false)`, which fails on a directory that still
holds the gitignored `.dumpcache.json`.

So a dump dir in that state is permanently unreachable by the sweep's own cleanup.

## Measured on the PDS mirror (2026-09-02, after a forced full `/Game` + `/App` sweep)

- 30,807 dirs with `meta.json`; 30,933 `.dumpcache.json` files.
- **126 dirs hold a `.dumpcache.json` and nothing else** — no `meta.json`, no
  sidecars. Zero dirs have the reverse shape.
- **121 of the 126 carry a current dumper fingerprint** —
  `pluginVersion 0.7.0` (matches `PinWright.uplugin`), `engineVersion 5.8.1-56057345`,
  current aspect versions. Only 5 are old (`0.4.0` / UE 5.7.4).
- The records assert files that do not exist. Example
  `asset-dumps/Game/DroneFootball/DA_PioneerSumo/.dumpcache.json` (written
  2026-08-17): `"writtenFiles": ["meta.json", "properties.json"]`,
  `"packageSavedHash": "24cd16be…"`, `"pluginVersion": "0.7.0"` — neither file is
  on disk. Most of the 126 are dead `/Game/DroneFootball/**` paths whose assets
  now live under `/App/App/**`.

## Why the stale record matters

`AssetDumpCache::IsCacheFresh` consults exactly that record. The dumper
fingerprint matches at HEAD, so freshness reduces to the source fingerprint; if
such a path is enumerated again on a **non-forced** sweep and its package hash
still matches, the sweep counts it `unchanged` and writes nothing, while the
mirror holds no dump for it. That is the `dumped`/`unchanged` split asserting
freshness for an asset the mirror does not contain — a reader trusting the
completion payload cannot tell it apart from a real cache hit
(`E-asset-dump-folder-cache-miss-reason-opaque` is the same blind spot from the
reporting side). The 2026-09-02 sweep ran with `force`, which bypasses the check,
so the defect did not bite that run.

The mechanism that removed the `meta.json` while leaving the cache file is **not
established** — the cache file is gitignored, so any out-of-band deletion of the
tracked mirror files (git operation, manual prune) produces this shape, and so
would a partial write. What is established is that once a dir is in it, nothing
in the tool gets it out.

## Fix

Two independent parts, either of which alone leaves a hole:

1. **Enumerate prune candidates by directory, not by `meta.json`.** A dump dir is
   any directory under `SweptRoot` holding a dump artifact — include
   `.dumpcache.json` in the recognition set so a record-only dir is a candidate
   and gets tree-deleted like any other dead dir.
2. **Treat a cache record whose `writtenFiles` are absent as stale.**
   `IsCacheFresh` already knows the file list; verifying existence before
   returning fresh closes the false-`unchanged` path regardless of how the dir
   got into that state, and costs one stat per listed file.

## See also
- `E-asset-dump-prune-empty-dirs` (DONE) — added the empty-dir pass; this is the
  adjacent case it cannot reach, because the dir is not empty.
- `B-dumpcache-staleness-loop` (IN-REVIEW) — the opposite shape (dump present,
  cache record missing).

## History
- `#1-initial-repro` `OPEN` reporter — `ReconcileMirrorSubtree` (`AssetDumpHandler.cpp:2274`) enumerates prune candidates via `FindFilesRecursive` for `meta.json` (`:2285-2288`), so a dump dir without one is never a candidate; the post-order empty-dir pass (`:2308-2324`) uses `DeleteDirectory(Tree=false)` and fails on a dir still holding the gitignored `.dumpcache.json`. Measured on the PDS mirror after the 2026-09-02 forced full sweep: 30,807 dirs with `meta.json` vs 30,933 `.dumpcache.json`; 126 dirs hold only the cache record, 0 the reverse; 121 of the 126 carry a current fingerprint (`pluginVersion 0.7.0`, `engineVersion 5.8.1-56057345`, current aspect versions) and assert `writtenFiles` that are not on disk — e.g. `asset-dumps/Game/DroneFootball/DA_PioneerSumo/.dumpcache.json` (written 2026-08-17) lists `meta.json` + `properties.json`, neither present; most are dead `/Game/DroneFootball/**` paths whose assets moved to `/App/App/**`. The route by which `meta.json` disappeared is not established and is not claimed. severity rationale: impact=Medium — the dirs are permanently unreachable by the tool's own cleanup, and the surviving record is what `IsCacheFresh` reads, so a later non-forced sweep over such a path can report `unchanged` for an asset the mirror does not hold; the wrong-`unchanged` outcome is conditional on re-enumeration (the 2026-09-02 sweep used `force`), which keeps it out of the High band where the caller is unconditionally handed a lie × reach=`asset.dump_folder` is common but record-only dirs are an edge within it, no modifier -> Medium. Fix: recognise `.dumpcache.json` as a dump artifact when enumerating prune candidates, and have `IsCacheFresh` verify the record's `writtenFiles` exist before returning fresh.
