---
id: B-dump-commit-window-leaves-cache-without-sidecars
title: "A cut-short asset dump can leave a dump dir holding only .dumpcache.json, its writtenFiles naming sidecars that are not on disk"
status: OPEN
severity: Low
category: bug
tags: [asset-dump, dump-folder, cache, dumpcache, atomicity, cancel]
encounters: 1
---

# A cut-short asset dump can leave a dump dir holding only .dumpcache.json, its writtenFiles naming sidecars that are not on disk

## Observed

In a project that keeps a **committed** dump mirror (`AssetDumpRootDirectory`
pointed at a tracked folder), 14 `FontFace` dump directories under two font
subtrees were found holding **only** `.dumpcache.json`. Their `meta.json` and
`properties.json` were gone from disk while the source `.uasset` files were
untouched, so version control reported 28 deleted files.

The surviving cache record was not corrupt or half-written — it was a complete,
internally valid record naming the two missing files:

```json
{
  "assetPath": "/<Root>/<Path>/<FontFace>",
  "cacheVersion": 2,
  "dumper": { "aspectVersions": { "meta.json": 6, "properties.json": 8 },
              "dumpCoreVersion": 3, "engineVersion": "5.8.2-0+UE5",
              "pluginVersion": "0.8.0" },
  "options": { "includeWidgetScreenshot": false },
  "source": { "diskSize": 24512, "fingerprintKind": "packageSavedHash",
              "packageExtension": "Asset",
              "packageSavedHash": "<40 hex>" },
  "writtenFiles": [ "meta.json", "properties.json" ]
}
```

It was byte-identical to the record a subsequent healthy dump of the same asset
produced, so every fingerprint in it (source hash, dumper, options) matched the
live asset. Only the files it claimed were absent.

## What is NOT broken — verified, do not "fix" this

The freshness path **already validates file existence**, so the mirror is not
poisoned permanently. `HasAllWrittenFiles`
(`Source\PinWright\Private\Handlers\Asset\AssetDumpCache.cpp:501`) stats every
entry of `writtenFiles` and returns `missing written file '<name>'`;
`IsCacheFresh` calls it at `:1183`, after the fingerprint comparisons and before
declaring the record fresh, and `IsDumpFresh` (`:1211`) is the only entry point
the two folder-sweep call sites use.

Live repro on UE 5.8 confirming the self-heal: one dump dir was put back into the
defect shape (valid `.dumpcache.json` restored, both sidecars removed), then
`asset.dump_folder` was run over its parent folder containing three assets. Result:

```
assetCount: 3, queued: 1, dumped: 1, unchanged: 2, skipCount: 0
```

The damaged asset was the one queued, and both sidecars came back byte-identical
to the pre-damage copies. A sweep therefore does **not** skip such a directory,
and adding a writtenFiles-existence check at the freshness gate would be
duplicate work.

## The actual gap

Nothing rebuilds or discards the cache marker when a dump's file transaction is
interrupted by **process death** rather than by an in-process error.

`WriteAssetDumpFiles` (`Source\PinWright\Private\Utils\AssetDumpWriter.cpp`) is a
stage/backup/commit/prune transaction:

1. `:533-567` — for every changed sidecar, `Move` the existing file out to
   `<BackupRoot>/changed/<name>`, then `Move` the staged replacement into place.
   Between those two moves the final path does not exist.
2. `:569-604` — prune files in the dump dir that the dump did not intend, by
   moving them to `<BackupRoot>/stale/`. Line `:584` **explicitly exempts**
   `.dumpcache.json` from that prune, so the cache marker of the previous run
   survives every step of the transaction.
3. `:620-621` — only on success are the stage and backup transaction directories
   removed.

`RollBack` (`:478-531`) restores both the backed-up originals and the removed
stale files, but it is an in-process lambda: it runs on a `Move`/`MakeDirectory`
failure and cannot run if the editor is killed, crashes, or is force-quit inside
the window. Since the marker is the one file deliberately kept across the whole
transaction, the residue of an interrupted rewrite is exactly the observed shape:
cache present, sidecars in a backup directory or gone, source asset unchanged.

The cache write itself is correctly ordered — `WriteDumpCacheForSuccessfulBaseline`
(`Source\PinWright\Private\Handlers\Asset\AssetDumpHandler.cpp:2421`, call site
`:2757`) runs after the file write reports success — so this is not a
write-before-durable ordering defect. It is the absence of any recovery for a
transaction that never reached step 3.

## Why it still matters at Low severity

For a `Saved/`-local mirror this is invisible: the next sweep repairs it. For a
**committed** mirror it surfaces as a block of unexplained deletions in version
control that an operator must diagnose by hand before committing, and which is
easy to mistake for a legitimate prune (an asset genuinely removed or merged
away). Nothing in the dump output says a previous transaction was interrupted.

## Suggested fixes (pick one, smallest first)

1. **Recover orphaned transaction directories at dump start.** If a
   `<StageRoot>`/`<BackupRoot>` for this dump dir already exists when a dump
   begins, a previous transaction died mid-flight: restore from `backup/` (or at
   minimum log it loudly) before starting the new one.
2. **Stop exempting the marker when the transaction is incomplete.** Treat
   `.dumpcache.json` as belonging to the transaction: write it last as today, but
   delete it first, so an interrupted rewrite leaves an *empty* dump dir whose
   state is unambiguous rather than a dir that looks dumped.
3. **Report it.** Have `asset.dump_folder` count directories it re-queued because
   of `missing written file` and surface that count (and the reason) in the
   completion payload, so a committed-mirror operator sees "N dumps recovered
   from an interrupted run" instead of silently correct output.

Fix 1 is the honest one; fix 3 is worth having regardless and pairs with
`E-asset-dump-folder-cache-miss-reason-opaque`.

## History

- `#1-filed-after-committed-mirror-gap` OPEN (Reporter) — Found 14 `FontFace`
  dump directories in a committed mirror holding only `.dumpcache.json` with
  valid fingerprints and absent sidecars. Verified on UE 5.8 that the freshness
  gate already rejects such a record (`missing written file`) and a sweep
  re-dumps it, so the mirror self-heals; the defect is the lack of recovery for a
  file transaction interrupted by process death, and the resulting unexplained
  deletions in a version-controlled mirror.
- `#2-observed-14-dirs-have-a-different-root-cause` OPEN (Reporter) — Correction to
  the provenance of `#1`: the 14 `FontFace` dump directories quoted there were not
  produced by an interrupted transaction. Each `Font` asset sat beside a same-named
  folder holding its `FontFace` assets, so the children's dump dirs are nested
  inside the parent asset's own dump dir, and the parent's recursive prune deleted
  their sidecars while the `.dumpcache.json` exemption kept their markers — see
  `B-asset-dump-dir-nested-inside-sibling-asset-dir`. The transaction-window gap
  analysed in this ticket is real and still worth hardening (nothing recovers an
  orphaned stage/backup root after process death); it simply is not what produced
  that observation, and this ticket's `encounters` should not be read as evidence
  for it.
