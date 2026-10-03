---
id: B-asset-dump-no-source-freshness-stamp
title: "An asset.dump mirror records nothing about the asset it mirrors — no source mtime, no package GUID — so a dump taken before a save still reads as authoritative after it and a reader re-files an already-fixed defect"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset-dump, umap, uworld, staleness, freshness, provenance, package-guid, mtime, silent-stale-data, weapons]
encounters: 1
lastSeen: 2026-09-07T07:26:00Z
---

# The dump is a mirror with no reference to what it mirrors

`asset.dump` writes a directory of sidecars that reads as a faithful, offline-inspectable copy of an
asset — and for a `.umap` that includes actor transforms, which is exactly the kind of fact a reviewer
reads out of the mirror rather than out of the editor. Nothing in the emitted tree names the state it
was taken from: no source `.uasset`/`.umap` modification time, no package GUID, no save counter, no
content hash of the source package. The dump's own file mtimes say when the dump was written; nothing
says *what* it was written from.

The consequence is that a dump does not become visibly wrong when the asset moves on. It stays
plausible, keeps its authoritative shape, and silently answers questions about a state that no longer
exists.

## Measured — WEAPONS critic review round 5, 2026-09-07

A `T_Weapons` dump taken at **07:06Z** still reports the witness actors `Pen_WoodWitness` and
`Pen_MetalWitness` at `(1180, -1080, 150)` / `(1180, +1080, 150)`. The map was saved at **07:26Z** with
both at `(1074, -1289, 150)` / `(1074, +1289, 150)` — a twenty-minute-old mirror describing a move
that had already happened.

A reviewer reading the dump today sees coordinates, a dump tree that looks complete, and no signal of
any kind that the numbers are twenty minutes stale. The natural outcome — and the one this round
came close to producing — is a **re-filed defect against an already-fixed asset**, argued from a
mirror that looks like evidence.

## Why the existing mechanisms do not cover this

- **`B-asset-dump-meta-missing-dumpedat` (WONTFIX)** declined a `dumpedAt` field on the reasoning that
  "filesystem mtime is the natural cache-freshness mechanism". That reasoning is about the **dump**
  side and is fine as far as it goes: the dump's mtime is on disk. It does not answer this ticket,
  because freshness is a *comparison* and the dump carries no identity for the other operand. The
  reader must already know which source file to `stat`, must think to do it, and gets nothing if the
  dump was copied, committed, or moved away from the asset. A recorded source mtime/GUID survives all
  three; a dump mtime alone survives none of them.
- **The plugin already owns a staleness detector.** `property.get` / `property.list` detect a stale
  dump today and emit a `hint` recommending a re-dump — see `E-property-stale-dump-hint-suggests-whole-project`
  (OPEN), which is a complaint about *what the hint recommends*, not about whether the detection
  exists. So the signal this ticket asks for is computed somewhere in the plugin already; it is simply
  not recorded in the dump tree and not surfaced to anyone reading the dump directly.

## What is asked for

1. **Stamp the source state into the dump.** In `meta.json`, record the source package's modification
   time and its package GUID (and/or a hash of the source package bytes). One field of provenance
   turns "is this current?" from an unanswerable question into a `stat` and a compare.
2. **Warn, or refuse, when the on-disk asset is newer than the dump.** Any verb that *reads* a dump on
   a caller's behalf should say so when the source has moved on; a bare `asset.dump` re-run should
   refresh rather than report success over a mirror it did not rewrite.
3. **Levels first if the work has to be staged.** A `.umap` dump is the worst case because actor
   transforms are the payload, level saves are frequent, and — unlike a Blueprint — a level's contents
   change with no compile step to prompt a re-dump.

## Severity

**Medium.** Impact is silent stale data on a normal path, which the board's own table places in the
High band — but held down one level because the correct value is trivially reachable (the live editor,
or a re-dump) and nothing is corrupted or lost by the dump being old. Held *up* from Low because the
failure produces confident wrong conclusions rather than friction, and because `asset.dump` output is
what offline review is built on in this project. A fixer who rates the level case High will get no
argument from this reporter.

## Related

- `B-asset-dump-meta-missing-dumpedat` (WONTFIX) — the dump-side timestamp, declined; this ticket is
  the source-side identity that the decline's own rationale presumes the reader already has.
- `E-property-stale-dump-hint-suggests-whole-project` (OPEN) — evidence that a stale-dump detector
  already exists on the property verbs, and the place its remedy text is wrong.
- `B-dumpcache-staleness-loop` (IN-REVIEW) — the sweep's `.dumpcache.json` freshness accounting;
  adjacent but internal to `dump_folder`, and invisible to a reader of the emitted tree.
- `F-dump-world-metadata` (DONE) — the ticket that made `.umap` dumps carry real content, which is
  what makes their staleness worth detecting.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Filed in a WEAPONS critic review round 5. Measured: a `T_Weapons` dump taken 07:06Z reports `Pen_WoodWitness` / `Pen_MetalWitness` at `(1180, -/+1080, 150)`; the map was saved at 07:26Z with both at `(1074, -/+1289, 150)`, so the mirror describes a pre-move state twenty minutes after the move, with nothing in the dump tree recording the source `.umap`'s mtime, package GUID, or a content hash. The mirror keeps its authoritative shape while being wrong, and the natural reader outcome is re-filing an already-fixed defect from it. Asks: stamp source mtime/GUID into `meta.json`; warn or refuse when the on-disk asset is newer than the dump; prioritise `.umap` because actor transforms are the payload and level saves carry no compile step to prompt a re-dump. Deliberately distinct from `B-asset-dump-meta-missing-dumpedat` (WONTFIX) — that declined a *dump-time* stamp because the dump's own filesystem mtime already carries it, which is true and does not help, since freshness is a comparison and the dump names no second operand; a recorded source mtime/GUID also survives the dump being copied, committed, or moved, which a bare dump mtime does not. Note for a fixer: the plugin already computes this signal somewhere — `property.get` / `property.list` emit a stale-dump `hint` today (`E-property-stale-dump-hint-suggests-whole-project`, OPEN, which disputes the hint's recommended remedy, not its existence) — so the ask is largely to record and surface what is already known rather than to build new detection. Severity Medium: impact class is silent stale data on a normal path (the High band), held one level down because the truth is a re-dump away and nothing is corrupted, held above Low because the failure produces confident wrong conclusions rather than friction. Searched the board for an owner before filing: `dumpedAt`, `mtime`, `freshness`, `stale`, `GUID`, `mirror`, `outdated` and `re-dump` across all 1866 tickets returned the four related tickets listed above and no ticket about the dump recording its source's state. No plugin source was opened for this ticket and no `file:line` is claimed.
- `#2-source-md5-stamp` `IN-REVIEW` developer — Still reproducible at plugin `10212ee4`: `AssetDumpBuilder::BuildMetaJson` emitted no source identity, the only source fingerprint (`packageSavedHash` / file stat) lived in the gitignored `.dumpcache.json`, and maps were `Uncached` there, so a committed level mirror had none at all. Fix, ask 1: `meta.json` gains `source: {fileMd5, unsavedChanges}` via new `AssetDumpBuilder::BuildSourceStampJson(PackageName)` (`Source/PinWright/Private/Utils/AssetDumpBuilder.{h,cpp}`). `fileMd5` = lowercase-hex MD5 of the package file on disk (`null` when the package has no file); `unsavedChanges` = the package was dirty, so the dump shows in-memory state the file does not hold. Deliberately **not** the asked-for mtime or package GUID: git checkout rewrites mtimes, so an mtime in a committed mirror would differ per machine and churn every sweep (the determinism this tree is built on), and the package GUID is deprecated in UE5 with nothing reliable to replace it. A content hash survives copy/commit/clone and is checkable with `md5sum` / `Get-FileHash -Algorithm MD5`, which is the "stat and a compare" the ticket asks for. Levels first (ask 3): an actor saved in its own package (one file per actor) keeps its transform in that package, not the `.umap`, so the WEAPONS case would not move the map hash; every external-reference entry in `actors/manifest.json` (loaded-actor and unloaded-descriptor branches in `Handlers/Asset/AssetDumpHandler.cpp`) now carries the same `source` stamp for its actor package. Ask 2: `asset.dump` already rewrites on every call (it never consults the cache), and no verb reads a dump on the caller's behalf; the one surface that judges a mirror is the stale-dump `hint` (`Utils/AssetDumpSuggestion.cpp`), whose level branch treated any existing mirror as current. It now compares `meta.json` `source.fileMd5` with the map file and says STALE with an `asset.dump` re-dump on a mismatch; a mirror without a stamp stays silent as before. Known ceiling (marked `ponytail:` at the site): the hint hashes the map file only, not every external actor package — a reader can still check those through the manifest stamps. Cache: `meta.json` aspect 6 -> 7, `actors/manifest.json` 1 -> 2 (`Handlers/Asset/AssetDumpCache.cpp`). The seeded dump-root `CLAUDE.md`/`AGENTS.md` staleness line names `source.fileMd5`. Docs: `docs/wiki-src/asset.dump-sidecars.md`, `CHANGELOG.md`. Tests: `PinWright.utils.asset_dump_builder.SourceStampHashesThePackageFile` (engine asset's `fileMd5` equals an independent MD5 of its resolved file; a dirty file-less package gives `null` + `unsavedChanges: true`; a `/Script` CDO still dumps with no warning) and `PinWright.AssetDumpSuggestion.LevelMirrorWithChangedMapFileIsStale` (mismatched recorded hash -> STALE hint naming `asset.dump`; matching stamp -> silent). Not covered by a test: the manifest wiring, since no OFPA world fixture exists in the suite.
- `#3-linux-verification` `IN-REVIEW` tester — Fix commit 383b0310. Passed non-skipped in run3/full: `PinWright.utils.asset_dump_builder.SourceStampHashesThePackageFile` (ask 1: `meta.json` `source.fileMd5` equals an independent MD5 of the package file, file-less dirty package gives null + `unsavedChanges`) and `PinWright.AssetDumpSuggestion.LevelMirrorWithChangedMapFileIsStale` (ask 2/3: a level mirror whose recorded hash no longer matches the map file gets a STALE hint naming `asset.dump`; matching stamp stays silent), plus the existing `ExistingLevelMirrorIsSilent`. Source read: `AssetDumpSuggestion.cpp` compares `source.fileMd5`. The MD5 instead of mtime/GUID is within the ticket's own "and/or a hash of the source package bytes". Remains: (1) the per-actor `source` stamps in `actors/manifest.json` for external (OFPA) actor packages have no test, as the developer states; (2) the stale hint hashes only the `.umap`, so on an OFPA level an actor move (the WEAPONS case, if that map is OFPA) does not trigger STALE, a `ponytail:` ceiling against ask 2. To close: dump a One-File-Per-Actor level and confirm every manifest entry carries `source.fileMd5`, and a human accepts or rejects the map-file-only hint ceiling.
