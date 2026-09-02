---
id: B-dump-reconcile-blind-to-metaless-dirs
title: "Mirror reconcile is keyed on meta.json, so a dump dir that keeps only its gitignored .dumpcache.json is unreachable by both prune passes and orphans forever"
status: OPEN
severity: Low
category: bug
tags: [asset-dump, dump-folder, reconcile, prune, dumpcache, cache, mirror]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# Mirror reconcile is blind to dump dirs that lost their `meta.json`

`ReconcileMirrorSubtree` (`Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp:2326`
at plugin HEAD `347826a6`; was `:2274` when filed)
finds prune candidates with a recursive `FindFilesRecursive` for `meta.json`
(`:2335-2338`) and tree-deletes each dir not in `LiveDirs`. A dump dir with **no**
`meta.json` is therefore never a candidate. The post-order second pass added by
`E-asset-dump-prune-empty-dirs` (`:2359-2378`) cannot finish the job either: it
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

## What the stale record does NOT do (corrected — see `#2`)

`#1` claimed the surviving record makes a later non-forced sweep report
`unchanged` for an asset the mirror does not hold. **That claim is wrong**, and it
was wrong when filed — not something an upstream change fixed.
`AssetDumpCache::IsCacheFresh` already verifies the record's `writtenFiles` are
on disk before returning fresh: `HasAllWrittenFiles`
(`Source/PinWright/Private/Handlers/Asset/AssetDumpCache.cpp:493-517`) stats every
listed file and returns `Stale("missing written file '<name>'")` on the first
absentee, and `IsCacheFresh` calls it as its last gate (`:1129-1132`), with
`HasOnlyListedDumpFiles` (`:1133-1136`) covering the reverse. The sweep reaches it
through `IsDumpFresh` (`:1191`), called at `AssetDumpHandler.cpp:1503` and `:2685`.
A record-only dir therefore always evaluates **stale**, and re-dumps. Fix part 2
below is consequently already implemented and is struck.

What remains is the prune blindness alone: the dirs are permanently unreachable by
the sweep's own cleanup and accumulate as orphans holding a gitignored file.

The mechanism that removed the `meta.json` while leaving the cache file is **not
established** — the cache file is gitignored, so any out-of-band deletion of the
tracked mirror files (git operation, manual prune) produces this shape, and so
would a partial write. What is established is that once a dir is in it, nothing
in the tool gets it out.

## Fix

One part remains:

1. **Enumerate prune candidates by directory, not by `meta.json`.** A dump dir is
   any directory under `SweptRoot` holding a dump artifact — include
   `.dumpcache.json` in the recognition set so a record-only dir is a candidate
   and gets tree-deleted like any other dead dir.
2. ~~**Treat a cache record whose `writtenFiles` are absent as stale.**~~
   **Already implemented** — `IsCacheFresh` -> `HasAllWrittenFiles`
   (`AssetDumpCache.cpp:493-517`, gate at `:1129-1132`). See `#2`.

## See also
- `E-asset-dump-prune-empty-dirs` (DONE) — added the empty-dir pass; this is the
  adjacent case it cannot reach, because the dir is not empty.
- `B-dumpcache-staleness-loop` (IN-REVIEW) — the opposite shape (dump present,
  cache record missing).

## History
- `#1-initial-repro` `OPEN` reporter — `ReconcileMirrorSubtree` (`AssetDumpHandler.cpp:2274`) enumerates prune candidates via `FindFilesRecursive` for `meta.json` (`:2285-2288`), so a dump dir without one is never a candidate; the post-order empty-dir pass (`:2308-2324`) uses `DeleteDirectory(Tree=false)` and fails on a dir still holding the gitignored `.dumpcache.json`. Measured on the PDS mirror after the 2026-09-02 forced full sweep: 30,807 dirs with `meta.json` vs 30,933 `.dumpcache.json`; 126 dirs hold only the cache record, 0 the reverse; 121 of the 126 carry a current fingerprint (`pluginVersion 0.7.0`, `engineVersion 5.8.1-56057345`, current aspect versions) and assert `writtenFiles` that are not on disk — e.g. `asset-dumps/Game/DroneFootball/DA_PioneerSumo/.dumpcache.json` (written 2026-08-17) lists `meta.json` + `properties.json`, neither present; most are dead `/Game/DroneFootball/**` paths whose assets moved to `/App/App/**`. The route by which `meta.json` disappeared is not established and is not claimed. severity rationale: impact=Medium — the dirs are permanently unreachable by the tool's own cleanup, and the surviving record is what `IsCacheFresh` reads, so a later non-forced sweep over such a path can report `unchanged` for an asset the mirror does not hold; the wrong-`unchanged` outcome is conditional on re-enumeration (the 2026-09-02 sweep used `force`), which keeps it out of the High band where the caller is unconditionally handed a lie × reach=`asset.dump_folder` is common but record-only dirs are an edge within it, no modifier -> Medium. Fix: recognise `.dumpcache.json` as a dump artifact when enumerating prune candidates, and have `IsCacheFresh` verify the record's `writtenFiles` exist before returning fresh.
- `#2-half-holds-half-was-never-true-severity-low` `OPEN` reporter — Re-checked against plugin HEAD `347826a6` after the 398-commit pull from `b16f0f2b`. **Split verdict.** Status stays `OPEN`, `encounters` unchanged (source re-read); **severity Medium -> Low**, and the body was corrected accordingly. (a) **The prune blindness still holds.** `ReconcileMirrorSubtree` moved `:2274` -> `AssetDumpHandler.cpp:2326` and is otherwise unchanged: still `FindFilesRecursive(AllMetaFiles, *SweptRoot, DumpFileNames::Meta, ...)` for the candidate set (`:2335-2338`), still `LiveDirs.Contains(DirPath)` tree-delete (`:2349-2353`), still a post-order `PostOrderPruneEmpty` calling `DeleteDirectory(*Dir, false, /*Tree=*/false)` (`:2359-2378`, was `:2308-2324`) which cannot remove a dir holding `.dumpcache.json`. Fix part 1 stands. (b) **The `IsCacheFresh` half was never a defect.** `IsCacheFresh` has always ended with `HasAllWrittenFiles(DumpDir, Record.WrittenFiles, Reason)` (`AssetDumpCache.cpp:1129-1132`) which stats each listed file and returns `Stale("missing written file '<name>'")` on the first absentee (`:493-517`), plus `HasOnlyListedDumpFiles` (`:1133-1136`); the sweep reaches it via `IsDumpFresh` (`:1191`) from `AssetDumpHandler.cpp:1503`/`:2685`. Verified this is **not** an upstream fix: `git show b16f0f2b:...AssetDumpCache.cpp` has byte-identical gates at the same lines (`HasAllWrittenFiles` at `:493`), so the claim was already false when `#1` was written on 2026-09-02 — reporter error, not a stale ticket. A record-only dir is therefore always `stale`, never `unchanged`, and the "reader trusting the completion payload is handed a lie" argument does not apply. severity rationale (re-scored): impact=Low — with the false-`unchanged` path removed, the residual is orphan directories the tool cannot clean up, each holding one gitignored file that git never sees; no wrong data, no blocked task, pure friction x reach=`asset.dump_folder` is common but record-only dirs are an edge within it, no modifier -> Low.
