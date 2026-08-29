---
id: B-asset-save-clobbers-out-of-band-change
title: "asset.save writes the resident in-memory package with no check that the .uasset changed on disk since it was loaded, so a git checkout, an external revert or another tool's write is silently overwritten by the next save of that asset"
status: OPEN
severity: High
category: bug
tags: [asset, save, in-memory-vs-disk, out-of-band, silent-overwrite, data-loss, stale-package, disk-state-ledger]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# `asset.save` has no idea what is on disk

`asset.save` (`Source/PinWright/Private/Handlers/Asset/AssetSaveHandler.cpp:41-131`) does
three things: `LoadObject` the path, optionally `MarkPackageDirty`, then
`SaveAssetToDiskReportingPresence(Asset, bForce, ...)`. For a package already resident in the
editor, `LoadObject` returns the in-memory object — it does not consult the file — and the
save then writes that in-memory state over whatever the `.uasset` currently holds.

There is no disk-state comparison anywhere on the path. `grep` over the handler for
`GetSavedHash`, `GetTimeStamp`, `FileSize` or any `IFileManager` probe returns nothing. The
handler cannot distinguish these two cases:

1. The resident package is the same revision that is on disk, plus the caller's edits. Saving
   is correct.
2. The `.uasset` on disk was replaced under the editor since the package was loaded or last
   saved — a `git checkout`, a manual revert, a rebase, another tool's write, a teammate's
   sync. The resident package is now a *fork* of an older revision, and saving discards the
   on-disk revision entirely.

Case 2 loses work that the editor never authored and never showed the caller. Nothing in the
response hints at it: `saved: true`, `sizeBytes`, done.

## Provenance

Scoped out deliberately by `B-source-control-revert-no-package-reload` (IN-REVIEW, High) `#6`,
whose fix made `source_control.revert` safe by resynchronising the loaded package through
`USourceControlHelpers::ApplyOperationAndReloadPackages`. Its closing paragraph:

> **Out of scope, needs its own ticket:** `asset.save` still writes the in-memory package
> unconditionally and does not detect that the `.uasset` on disk changed under it since the
> package was loaded/saved, so any OUT-OF-BAND working-tree change (a `git checkout`, an
> external revert, another tool's write) is still silently overwritten by the next save of a
> resident package. Reverting through `source_control.revert` is now safe because it
> resynchronizes; reverting behind the editor's back is not. A fix needs a per-package "disk
> state at last load/save" ledger (mtime+size, or `UPackage::GetSavedHash()` vs the file) that
> does not exist today, and would have to decide refuse-vs-warn — a separate design, not a
> line in this handler.

Re-verified in the current tree: `AssetSaveHandler.cpp` is unchanged in this respect and the
ledger does not exist.

The narrowed hazard after that fix is exactly this: the *in-editor* revert path is now safe,
which makes the *out-of-editor* path the remaining one — and it is the one a developer or a
parallel agent reaches for most often. Every fix host in this workflow does
`git reset --hard` + `git clean -fd` between iterations while an editor may still be resident.

## Why this is not covered by the two nearest tickets

- `E-asset-save-force-no-clean-check` (OPEN, Medium, `blockedBy: [F-asset-save]`) asks for a
  cheap *already-clean* read so callers stop issuing defensive force-saves. That is about
  wasted calls; this is about a write that destroys the file it lands on. Same verb, opposite
  direction — and the readback that ticket asks for would be a natural place to publish this
  one's answer.
- `B-bp-saved-state-corruption-mcp-edits` covers corruption of the Blueprint's own saved state
  by the edit path, not a stale-vs-disk race.

## Severity

**High.** Impact class sits between the rubric's Critical ("a write that ... loses asset
data") and High ("silent false-success ... the caller trusts a result that is a lie"). Taking
High rather than Critical, argued:

- The write itself is not corrupt — it produces a valid, loadable `.uasset`. What is lost is
  the *other* revision, and it is lost silently.
- The loss is recoverable exactly when the discarded revision was tracked, which is the common
  case for the scenario that triggers it (a `git checkout` implies git). It is unrecoverable
  when it is not.
- Reach: `asset.save` runs in nearly every authoring session, which argues a bump up; the
  precondition (an out-of-band write to a package that is still resident) is uncommon, which
  argues a bump down. Recorded as cancelling rather than either applied silently.

Held level with `B-source-control-revert-no-package-reload`, which is High for the same class
in its in-editor form.

**Workaround:** never modify the working tree under a live editor. If you must, run
`asset.reload` (or `source_control.revert`, which now resynchronises) on every touched package
before the next save, or quit the editor first. Do not rely on `asset.save` noticing.

**Fix, and the open design question this ticket exists to settle:** maintain a per-package
disk-state record at load and at each successful save — either `mtime` + size, or
`UPackage::GetSavedHash()` compared against the file — and consult it before writing. Then
decide **refuse vs warn**, which is the part that needs a decision and not just code:

- *Refuse* (`SAVE_DISK_STATE_DIVERGED` + the two states, caller re-reads or forces) is safe by
  default and matches the plugin's refuse-to-write convention on the Blueprint integrity gate
  in this same handler, but breaks any workflow that legitimately expects last-write-wins.
- *Warn* (save, and report `diskStateDiverged: true` with both fingerprints) never blocks, but
  a warning in a response that already says `saved: true` is exactly the shape the board keeps
  filing tickets about.

A `force: true` that already means "bypass the throttle" should probably not silently acquire
"and bypass the divergence gate" — that overloading needs deciding too.

## History
- `#1-out-of-band-write-silently-clobbered` `OPEN` reporter — Source-only; **no editor call, no save, and no working-tree experiment was run this pass** — the tree is mid-verification on another wave and the reproduction (edit an asset, `git checkout` the `.uasset` under the live editor, `asset.save`, inspect the file) was not performed. What is established is the mechanism, read from the whole handler: `AssetSaveHandler.cpp:58` `LoadObject` returns the resident object for an already-loaded package; `:66-69` optionally marks it dirty; `:93` calls `SaveAssetToDiskReportingPresence`; and `grep` over the file for `GetSavedHash`, `GetTimeStamp`, `FileSize` and `IFileManager` returns nothing, so no disk state is consulted at any point. The only gate on the path is the Blueprint integrity check (`:77-83`), which inspects the in-memory graph, not the file. Filed on the explicit instruction of `B-source-control-revert-no-package-reload` `#6`, which scoped this out as needing its own ticket and its own refuse-vs-warn decision; that ticket's own fix narrowed the hazard to the out-of-editor path, which is the one the fix-workflow hosts hit every iteration (`git reset --hard` + `git clean -fd` between runs, per `B-tests-leak-host-content`'s workaround note). Dedup: board greps for `out-of-band`, `git checkout` and `GetSavedHash` surface only that parent, `B-bp-saved-state-corruption-mcp-edits` (different mechanism — corruption by the edit path, not a stale-vs-disk race) and `E-asset-save-force-no-clean-check` (same verb, opposite direction: wasted calls, not a destructive write). Severity High, argued above against Critical; the refuse-vs-warn choice is left open on purpose and is the substance of the fix.
