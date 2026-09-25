---
id: B-asset-dump-dir-nested-inside-sibling-asset-dir
title: "An asset whose name equals a sibling folder's name gets a dump dir containing that folder's dump dirs — non-idempotent sweeps and sidecar deletion"
status: IN-REVIEW
severity: High
category: bug
tags: [asset-dump, dump-folder, dir-resolution, dumpcache, prune, data-loss, idempotence]
encounters: 1
---

# An asset whose name equals a sibling folder's name gets a dump dir containing that folder's dump dirs

## Layout that triggers it

A content layout where an asset's package name is identical to a sibling
folder's name — e.g. a `Font` asset `X.uasset` sitting beside a folder `X/`
that holds its `FontFace` assets. This is a normal, engine-produced layout
(importing a font emits exactly this shape), not a pathological one.

`ResolveDumpDir` (`Source/PinWright/Private/Utils/AssetDumpWriter.cpp:40`)
derives the dump directory purely from the package path — strip the leading
`/`, join under the root. So the asset `/<Root>/<Path>/X` and the folder
`/<Root>/<Path>/X/` map to the **same** mirror directory, and every asset
inside that folder maps to a directory **nested inside the parent asset's own
dump dir**:

```
<mirror>/<Root>/<Path>/X/                 <- the Font asset's dump dir
<mirror>/<Root>/<Path>/X/meta.json                 (parent's sidecars)
<mirror>/<Root>/<Path>/X/properties.json
<mirror>/<Root>/<Path>/X/X_Regular/       <- a child FontFace's dump dir
<mirror>/<Root>/<Path>/X/X_Regular/meta.json
<mirror>/<Root>/<Path>/X/X_Bold/...
```

Nothing in the dump-dir contract says a dump dir is a leaf, and both the
freshness check and the writer's prune walk the dump dir **recursively**, so
they treat the children's files as the parent's.

## Consequences (all four observed on one sweep pair)

**1. The sweep is not idempotent.** `HasOnlyListedDumpFiles`
(`Source/PinWright/Private/Handlers/Asset/AssetDumpCache.cpp:527`, called from
`IsCacheFresh` at `:1187`) does a `FindFilesRecursive` over the dump dir and
rejects any file not in the record's `writtenFiles`. Every nested child file
therefore reads as foreign, and the parent's cache is declared stale on every
run:

```
asset.dump cache: stale dump for '<parent>' (dumpDir='...'): unlisted dump file '<Child>/.dumpcache.json'
```

The parent is re-dumped on every sweep, forever — a `dump_folder` over such a
subtree never reaches `unchanged`.

**2. The parent's prune deletes the children's sidecars.** `WriteAssetDumpFiles`
(`Source/PinWright/Private/Utils/AssetDumpWriter.cpp`, prune loop at `:569-604`)
also enumerates with `FindFilesRecursive` (`:576`) and moves every file that is
not one of the parent's own intended outputs into the transaction's `stale/`
backup, which is deleted on success (`:620-621`). The one exemption at `:583` is
a clean-filename match on `.dumpcache.json`, so each nested child keeps its cache
marker and loses its `meta.json` and `properties.json`. Observed: two such font
subtrees, 14 child dirs, **28 deleted sidecars**, source assets untouched.

**3. The mirror flip-flops.** On the next sweep each stripped child fails its own
`HasAllWrittenFiles` (`AssetDumpCache.cpp:501`) with `missing written file`, gets
re-queued and re-emits both sidecars — and the parent's prune deletes them again.
The mirror therefore oscillates between two states, and every sweep produces a
spurious diff of the same 28 files.

**4. In a committed mirror it looks like a cut-short run.** Version control shows
a block of deletions of files nobody touched, under directories that still hold a
valid-looking `.dumpcache.json`. That is exactly the shape
`B-dump-commit-window-leaves-cache-without-sidecars` describes, and the 14-dir
observation recorded there was initially misdiagnosed as an interrupted file
transaction. The transaction-window analysis in that ticket is still valid
hardening on its own merits, but it is **not** the cause of that observation —
this ticket is.

`ReconcileMirrorSubtree` (`Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp:2774`)
keys on `meta.json` locations, so it sees parent and children as separate live
dirs and does not itself delete anything here — but it also cannot tell that a
child dir is nested inside a live parent dir, so any future change to its
liveness set inherits the same ambiguity.

## Root cause

The mirror path is not injective: an asset path and a folder path with the same
spelling collapse to one directory. Both recursive walks
(`HasOnlyListedDumpFiles`, the writer's prune) then assume everything below a
dump dir belongs to that one asset.

## Candidate fixes

**(a) Make an asset's dump dir unambiguous.** Give asset dump dirs a suffix or a
leaf marker so they can never equal a folder's dir — e.g. `<Name>.<assetclass>/`,
or an `__asset` leaf under `<Name>/`. This removes the ambiguity rather than
tolerating it, and reconciles with the `<Name>/` convention settled in
`B-asset-dump-dir-naming-inconsistent`. It changes every mirror path, so it needs
an aspect-version-style migration note (and an existing committed mirror has to
be regenerated or migrated once).

**(b) Make the recursive walks ignore other assets' dirs.** In both
`HasOnlyListedDumpFiles` and the writer's prune, skip any subdirectory that
contains its own `.dumpcache.json` — such a directory belongs to a different
asset. Smaller change, no path migration, and it stops the data loss and the
flip-flop immediately; the nesting itself remains.

(b) is the smaller change and the one that stops the deletion; (a) is the one
that removes the underlying ambiguity. They are not exclusive — (b) now, (a) when
a mirror regeneration is acceptable.

Either fix wants a regression test over a synthetic fixture of the same shape
(one asset plus a same-named folder holding a second asset), asserting that a
second sweep reports the parent `unchanged` and that the child's sidecars survive.

## History

- `#1-nested-dump-dirs-eat-child-sidecars` `OPEN` reporter — Found during a dump
  reconcile of a committed mirror on Linux/UE 5.8: two `Font` assets each sitting
  beside a same-named folder of 14 `FontFace` assets produced dump dirs that
  physically contain their children's dump dirs. Parent freshness fails every
  sweep with `unlisted dump file '<Child>/.dumpcache.json'`, the parent's prune
  removes the 28 nested `meta.json`/`properties.json` files (the `.dumpcache.json`
  exemption is a clean-filename match, so the children's markers survive), and the
  next sweep re-emits them — the mirror oscillates and every sweep shows the same
  spurious diff. Root cause is `ResolveDumpDir` deriving the dir from the package
  path alone, so an asset path and a same-spelled folder path collapse to one
  directory, combined with the two `FindFilesRecursive` walks that assume a dump
  dir is a leaf. Supersedes the interrupted-transaction diagnosis recorded in
  `B-dump-commit-window-leaves-cache-without-sidecars` `#1` for that observation.
- `#2-own-files-walk-skips-nested-dump-dirs` `IN-REVIEW` developer — Took fix (b),
  applied at every walk that treats a dump dir's contents as one asset's, through one
  shared helper: `AssetDumpWriter::FindOwnDumpFiles` (`Utils/AssetDumpWriter.h/.cpp`)
  enumerates a dump dir recursively but does not descend into a subdirectory holding its
  own `meta.json` or `.dumpcache.json` (another asset's dump dir; `meta.json` is included
  so a child whose cache record was never written is still protected, `.dumpcache.json`
  so a child already stripped by the old prune is too). Callers switched from
  `FindFilesRecursive`: `HasOnlyListedDumpFiles` (`AssetDumpCache.cpp`, consequence 1),
  the writer's stale-sidecar prune (`AssetDumpWriter.cpp`, consequences 2-4), and
  `ReconcileMirrorSubtree` (`AssetDumpHandler.cpp`), which used to tree-delete a dead dump
  dir and so would have wiped every live nested dump when the parent asset (e.g. the Font)
  is deleted but its folder is kept; it now deletes the dead dir's own files and leaves
  emptied dirs to the existing post-order pass. Fix (a) (unambiguous dir naming) was
  deliberately not taken: it moves every mirror path to fix a shape that occurs in 2 of
  30,904 dump dirs of the host mirror, and forces a one-time regeneration of every
  committed mirror. With (b) the layout is unchanged, so no migration: an already-damaged
  mirror self-heals on the next sweep (stripped children re-dump once, parent now reads
  fresh). Documented in `docs/wiki-src/asset.dump-quickstart.md` (layout paragraph).
  Out of scope, same non-injective path: diff dirs under `asset-dump-diffs/` still nest,
  and a parent's diff-dir `Tree=true` cleanup wipes its children's diff artifacts
  (transient, Saved-only). Tests (all new):
  `PinWright.utils.asset_dump_writer.PruneSkipsNestedAssetDumpDirs`,
  `PinWright.AssetDumpCache.NestedAssetDumpDirIsNotUnlisted` (also asserts an unmarked
  subdir file still reads unlisted),
  `PinWright.asset.dump.FolderReconcileKeepsLiveDumpDirNestedInDeadParent`,
  `PinWright.asset.dump.AsyncFolderDump.AssetBesideSameNamedFolderIsIdempotent` (end to
  end: two saved MIC fixtures `<F>/Nested` + `<F>/Nested/Nested_Child` under
  `/Game/PinWrightTests`, two recursive sweeps plus a direct parent re-dump). Baseline on
  unfixed code: all 4 red, reproducing the ticket verbatim (second sweep queued 1 /
  unchanged 1, log `stale dump for '.../Nested' ...: unlisted dump file
  'Nested_Child/.dumpcache.json'`, child sidecars deleted by both the sweep and the
  direct re-dump). With the fix: 4/4 green; scoped run
  `PinWright.utils.asset_dump_writer+PinWright.AssetDumpCache+PinWright.asset.dump` =
  106 performed, 105 pass, 1 fail = the pre-existing
  `B-commit-failure-rollback-test-red-on-linux` (expected-error declaration, unrelated),
  1 pre-existing skip (`AnimBlueprint_EmitsAnimGraphJson`). Plugin commit `f1a04a23`.
