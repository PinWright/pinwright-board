---
id: B-asset-delete-force-delete-leaves-uasset-on-disk
title: "asset.delete force-deletes the objects out of memory but never removes the .uasset: UEditorAssetLibrary::DeleteAsset returns true, the file's mtime never moves, and an unreferenced asset is undeletable through the plugin"
status: OPEN
severity: High
category: bug
tags: [asset, asset-delete, force-delete, editor-asset-library, objecttools, disk-vs-memory, no-workaround, cleanup, shared-editor]
encounters: 2
lastSeen: 2026-08-27T20:46:02+05:00
---

# `asset.delete` cannot delete an asset that nothing references

`asset.delete` reports `deleteReported: true` per entry — `UEditorAssetLibrary::DeleteAsset`
returned `true` — and then its own post-check finds the asset still there:

```json
{"success":false,"deletedCount":0,"requestedCount":3,"failedCount":3,"missingCount":0,
 "existsAfter":true,
 "results":[{"path":"/Game/Atlantis/VFX/NE_Bubbles_Stream","kind":"asset","existedBefore":true,
             "deleteReported":true,"existsAfter":true,"deleted":false,"missing":false}, ...],
 "deleted":[],"failed":["/Game/Atlantis/VFX/NE_Bubbles_Stream",
                        "/Game/Atlantis/VFX/NE_FishSchool","/Game/Atlantis/VFX/NE_Fish_A"],
 "failureHint":"One or more paths still exist after the delete. The usual causes are an open asset
   editor, a read-only file, or a live reference (see referencingBlueprints). This verb does not
   close editors or force-GC on your behalf.",
 "referencingBlueprints":[]}
```

**All three causes the `failureHint` names are ruled out** (evidence below), so the hint sends the
caller after things that are not the problem and there is no next step it offers that works.

The verb's *reporting* is correct and is the only reason this was caught — `existsAfter` is exactly
the disk-side verification this project's host `CLAUDE.md` § *"Verify a write against disk, not
against the object you just wrote"* asks for, and it did its job. The defect is underneath it: the
delete does not delete.

## Repro (deterministic, reproduced twice by two agents)

```
asset.delete { paths: ["/Game/Atlantis/VFX/NE_Bubbles_Stream",
                       "/Game/Atlantis/VFX/NE_FishSchool",
                       "/Game/Atlantis/VFX/NE_Fish_A"] }
```

Three orphan `UNiagaraEmitter` assets left over from an abandoned from-scratch authoring approach.
Response as above; assets still present afterwards.

## The objects ARE deleted from memory — only the file survives

`Saved/Logs/EAContentExamples58.log`, at `2026.08.27-15.46.02` UTC (20:46 local; logs are UTC+0,
machine UTC+5), once per asset and **with no error, warning or source-control line of any kind**:

```
[15.46.02:334][483]LogStreaming: Display: FlushAsyncLoading(401): 1 QueuedPackages, 0 AsyncPackages
[15.46.02:334][483]LogUObjectGlobals: Force Deleting 1 Package(s):
	Asset Name: /Game/Atlantis/VFX/NE_Bubbles_Stream.NE_Bubbles_Stream
	Asset Type: NiagaraEmitter
[15.46.02:560][483]LogUObjectHash: Compacting FUObjectHashTables data took   0.46ms
[15.46.02:576][483]LogUObjectGlobals: Force Deleting 1 Package(s):
	Asset Name: /Game/Atlantis/VFX/NE_FishSchool.NE_FishSchool
[15.46.02:810][483]LogUObjectGlobals: Force Deleting 1 Package(s):
	Asset Name: /Game/Atlantis/VFX/NE_Fish_A.NE_Fish_A
```

`Force Deleting N Package(s)` is `ObjectTools::ForceDeleteObjects`. So the call reached the force
path and the in-memory objects went away — but the packages were never unloaded-and-unlinked from
disk. **The three `.uasset` files still carry their pre-delete mtimes**, i.e. nothing touched them:

```
NE_Bubbles_Stream.uasset   34757 B   2026-08-27 13:01:39
NE_FishSchool.uasset       34737 B   2026-08-27 15:04:33
NE_Fish_A.uasset          228881 B   2026-08-27 18:33:28
```

That is the whole defect in one line: **memory says deleted, disk says present.** Until the editor
restarts, the two disagree.

## Every cause the `failureHint` names, ruled out

| `failureHint` cause | checked how | result |
|---|---|---|
| open asset editor | `grep -c "Opening Asset editor for"` over the live log | **0** for the whole session |
| read-only file | `Get-ChildItem \| Select Attributes, IsReadOnly` | `Archive`, `IsReadOnly False` — all three writable |
| live reference | `asset.dependencies` per asset | `referencerCount: 0`, `referencers: []` — all three |
| live reference | `referencingBlueprints` in the delete response | `[]` |

Cross-checked from the other direction as well: `asset.references` on the shipping
`/Game/Atlantis/VFX/NS_FishSchool` lists 67 outbound dependencies, and the only `/Game` entries are
`/Game/Atlantis/Materials/MI_Fish_Blue` and `/Game/Atlantis/Meshes/SM_Fish_A` — **no `NE_*`**. The
shipping systems were rebuilt from stock template duplicates and genuinely do not reference these
emitters. There is nothing for the caller to release.

No source-control provider is configured either (`LogSourceControl: Uncontrolled asset discovery
finished ... Found 14241 uncontrolled assets`; no `SourceControlSettings.ini`), so the file delete
should be taking the plain `IFileManager::Delete` route with nothing in the way.

## Guilty source

`Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp`, the `asset.delete`
handler registered at `:518`:

```cpp
585:        const bool bExistedBefore = bIsDirectory || UEditorAssetLibrary::DoesAssetExist(Path);
588:        const bool bDeleteReported = bExistedBefore
589:            && (bIsDirectory ? UEditorAssetLibrary::DeleteDirectory(Path)
                                 : UEditorAssetLibrary::DeleteAsset(Path));
603:        bStillExists = ...   // the post-check that catches this
```

`UEditorAssetLibrary::DeleteAsset` -> `ObjectTools::DeleteAssets(..., bShowConfirmation=false)` ->
`ObjectTools::ForceDeleteObjects` -> `ObjectTools::CleanupAfterSuccessfulDelete`.

**Unproven hypothesis, stated as such:** `CleanupAfterSuccessfulDelete` performs its own reference
check and *silently skips* unloading the package and deleting its file when any in-memory referencer
remains, while `ForceDeleteObjects` has already logged `Force Deleting` and torn the objects down.
That would produce exactly this signature — force-delete logged, no error, file untouched. Two
candidate holders, neither confirmed:

- the **transaction/undo buffer** (`UTransBuffer`), which this host's `CLAUDE.md` already documents
  as holding stale references across `REINST_` recompiles;
- **PinWright's own cached Niagara system view models** in `FPluginState`
  (`PinWrightNiagara::AcquireSystemViewModel`) — if one was acquired for a system that once
  contained these emitters, the plugin itself is the referencer blocking the plugin's own delete.

A fixer should confirm which (if either) before choosing a fix; do not implement against this
paragraph as if it were established.

## Impact

High. `asset.delete` is a routine cleanup verb and this is the ordinary case for it — an orphan with
no referencers at all. There is **no workaround inside the plugin**: the caller has done everything
the error hint asks and the asset is still there, so dead content cannot be removed and ships.

Severity rationale: impact = hard blocker with no workaround on a normal path x reach = a verb used
in most build/cleanup sessions -> High.

**A fixer should consider re-rating this Critical** if they confirm the first half of the hypothesis
above. `ForceDeleteObjects` nulls out references to the deleted objects in every referencing object
*before* the file delete is attempted. In the observed case that was harmless because referencer
count was genuinely 0 — but on an asset that *does* have referencers, a force-delete that nulls
those pointers and then fails to remove the file leaves referencing assets silently pointing at
nothing while the target still exists on disk. If any of them is then saved, that is data loss from
a call that reported failure. That path was not exercised here and is not claimed as observed.

## What it should do

1. **Do not report `deleteReported: true` for a delete that did not delete.** `existsAfter` already
   catches it; propagate that into the per-entry verdict instead of recording the engine's return
   value as if it were the outcome.
2. **Say what actually blocked it.** `referencingBlueprints: []` is answering a narrower question
   than the caller asked — the holder here is not a blueprint. Enumerate real in-memory referencers
   (`FReferencerInformation` / `FArchiveFindCulprit`, which is what `ObjectTools` itself uses for the
   confirmation dialog) and name them in the error, per `agent-conventions.md` § *Error conventions*
   ("what was searched, what class/path was checked, and an actionable hint").
3. **Rewrite the `failureHint`.** As written it lists three causes that were all false here, which
   costs the caller a full investigation before they learn the hint was not applicable.
4. **Decide and document the memory/disk divergence.** Either do not force-delete until the file
   delete is known to be possible, or report explicitly that the objects were torn down while the
   files remain, so the caller knows the editor's view and disk have diverged.
5. Confirm whether `asset.bulk_delete` shares the code path (see below) — if it does, one fix covers
   both; if it does not, the two verbs silently differ in whether cleanup works.

## Not tried, deliberately

- **`asset.bulk_delete`** is the obvious next verb and was **not** run. Its documented redirector
  fixup is scoped to the deleted assets' own folders, i.e. `/Game/Atlantis/VFX/`, and a VFX
  placement agent was actively working in that exact folder in this shared editor at the time
  (`actor.set_transform` / `actor.set_label` on `VFX_BubblesAmbient_*` and `VFX_BubbleVent_*` in the
  live log). Worth trying on a quiet editor to establish whether the two verbs diverge.
- **Deleting the `.uasset` files from the filesystem** was not done. It would leave the asset
  registry and any redirectors inconsistent, and it would hide this defect rather than record it.

## History

- `#1-two-attempts-one-session` `OPEN` reporter — UE 5.8, `EAContentExamples58`,
  `/Game/Maps/Atlantis`, shared editor. Observed twice in one session on the same assets. First
  attempt by another agent at `2026.08.27-15.34.32` UTC on `/Game/Atlantis/VFX/NE_Fish_A` alone
  (`Saved/Logs/EAContentExamples58.log:2686`, `Force Deleting 1 Package(s)`, no error, file
  survived); second by this reporter at `15.46.02` UTC on all three (`:2852`, `:2857`, `:2862`),
  identical outcome. Unreferencedness established before deleting, not assumed: `asset.dependencies`
  returned `referencerCount: 0` for each, and `asset.references` on the shipping `NS_FishSchool`
  showed its only `/Game` dependencies are `MI_Fish_Blue` and `SM_Fish_A`. Not traced into
  `ObjectTools` beyond identifying `UEditorAssetLibrary::DeleteAsset` at
  `AssetManageHandler.cpp:589` as the call and `ForceDeleteObjects` as the reached engine path — the
  `CleanupAfterSuccessfulDelete` hypothesis in `## Guilty source` is inference from the log
  signature, not read from a debugger.
