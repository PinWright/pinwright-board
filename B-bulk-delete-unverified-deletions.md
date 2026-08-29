---
id: B-bulk-delete-unverified-deletions
title: "asset.bulk_delete verifies nothing: deleted[] is the requested list, success is DeletedCount > 0, and a partially-deleted batch reports as a clean success"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, bulk-delete, asset-delete, objecttools, false-success, unverified-write, disk-vs-memory, partial-batch, whole-batch-abort, attribution]
encounters: 2
lastSeen: 2026-08-29T17:05:00+05:00
---

# `asset.bulk_delete` reports the request back as the result

`Handlers/Asset/AssetWorkflowHandler.cpp`, the `asset.bulk_delete` handler, builds its
entire response from the inputs and one integer:

```cpp
int32 DeletedCount = ObjectTools::DeleteObjects(ObjectsToDelete, bShowConfirmation);
...
for (const FString& Path : ValidPaths) { DeletedArray.Add(MakeShared<FJsonValueString>(Path)); }
Result->SetBoolField(TEXT("success"), DeletedCount > 0);
Result->SetArrayField(TEXT("deleted"), DeletedArray);
Result->SetNumberField(TEXT("requested"), ObjectsToDelete.Num());
```

`deleted[]` is `ValidPaths` — the caller's own paths, echoed back, populated **before**
the delete ran and never touched afterwards. `success` is `DeletedCount > 0`. There is no
asset-registry probe and no on-disk probe anywhere in the handler. The only per-path
information the engine returns is a **count**, so a partial batch is not merely unreported,
it is unrepresentable: delete 1 of 20 and the response is `success: true` with all 20 names
sitting in `deleted[]`.

The sibling verb `asset.delete` had the identical shape and was fixed
(`B-asset-delete-force-delete-leaves-uasset-on-disk`, IN-REVIEW); that fix explicitly left
this one open. `Docs/wiki-src/asset.md` § *asset.bulk_delete* already documents the defect
("**It does not verify its deletions.**") but no ticket existed.

## Engine mechanism

The two verbs reach `CleanupAfterSuccessfulDelete` by **different** routes, and the
difference matters for the fix:

- `asset.delete` → `UEditorAssetSubsystem::DeleteAsset` → `ObjectTools::ForceDeleteObjects`
  → `CleanupAfterSuccessfulDelete(Packages)` — reference check **on**.
- `asset.bulk_delete` → `ObjectTools::DeleteObjects` → `ObjectTools::DeleteItems` →
  `FAssetDeleteModel::DoDelete` (`AssetDeleteModel.cpp`) → `ObjectTools::DeleteObjectsUnchecked`
  → `CleanupAfterSuccessfulDelete(Packages, bPerformReferenceCheck=false)`.

**Correction to the lead this was filed from:** the `GatherObjectReferencersForDeletion` cull
does *not* fire on the bulk path — `DeleteObjectsUnchecked` passes `bPerformReferenceCheck=false`,
so `bIsReferenced` stays false. A referenced asset is instead rejected one level up:
`FAssetDeleteModel::CanDelete()` is `!CanForceDelete()`, and `CanForceDelete()` is true when
anything in the batch is referenced in memory by non-undo or has on-disk referencers — so
`DoDelete()` returns false for the **whole batch**, `ObjectsDeleted` stays 0, and the handler
does correctly error. That case is honest by accident. Every other failure mode is not:

1. **Read-only package, source control disabled.** `DeleteObjectsUnchecked` calls
   `ObjectTools::MakeReadOnlyPackageWritable` per object; it opens an `FMessageDialog`, which
   returns its `No` default unattended, and the object is `continue`d — not deleted, not
   counted, **not logged**. The other objects in the batch delete normally, so
   `DeletedCount > 0` and the response claims the read-only one went too.
2. **Delegate veto.** `ObjectTools::DeleteSingleObject` returns false when an
   `FEditorDelegates::OnAssetsCanDelete` handler refuses; the only signal is a modal dialog
   nothing sees unattended. Same silent skip.
3. **Filename unresolvable.** In `CleanupAfterSuccessfulDelete`, a package for which
   `FPackageName::DoesPackageExist` cannot produce a filename is dropped from
   `PackagesToDelete` into `PackagesToUnload`: it is unloaded from memory and its file is
   left on disk, with no log. Same for a non-`UPackage` entry. This runs **after** the count
   is already final.
4. **The file delete itself is unchecked.** `CleanupAfterSuccessfulDelete` calls
   `IFileManager::Get().Delete(*PackageFilename)` and **discards the return value**; SCC
   revert/delete failures only `UE_LOG(..., Warning)`; and a read-only file with SCC disabled
   hits `FMessageDialog` again, leaving `bDeletedFileLocallyWritable` false so the file is
   never touched. Every one of these leaves a `.uasset` on disk that the count already scored
   as deleted — the exact memory/disk divergence observed live on the sibling verb.
5. **The count is not even bounded by the request.** `ObjectTools::AddExtraObjectsToDelete`
   (called by both `DeleteItems` and `DeleteObjectsUnchecked`) appends whatever
   `FEditorDelegates::OnAddExtraObjectsToDelete` broadcasts plus every external package of
   each target's outer package. On an OFPA/external-package asset the count can be positive
   with none of the caller's named assets gone, and `success: true` follows.

## Second defect in the same block: dropped paths are never reported

```cpp
for (const FString& AssetPath : AssetPaths)
    if (UEditorAssetLibrary::DoesAssetExist(AssetPath))
        if (UObject* Asset = UEditorAssetLibrary::LoadAsset(AssetPath))
            { ObjectsToDelete.Add(Asset); ValidPaths.Add(AssetPath); }
```

A path that does not exist, or exists but fails to load, is silently dropped from *both*
`ValidPaths` and `ObjectsToDelete`. `requested` is then the count that **loaded**, not the
count the caller asked for, and there is no `missing[]` or `failed[]` array — so the caller
has no field that can tell them a path was never attempted. `asset.delete` reports exactly
this case as `missing`.

## Repro

Source-derived, not executed (this ticket is a read of current source; the same defect class
was observed live on the sibling verb — see `B-asset-delete-force-delete-leaves-uasset-on-disk`
for the log evidence). Two cheap deterministic checks, both on a scratch folder:

1. **Dropped path.** `asset.bulk_delete { assetPaths: ["/Game/Scratch/SM_Real",
   "/Game/Scratch/SM_Typo"] }` → `success: true`, `requested: 1`, `deleted: ["/Game/Scratch/SM_Real"]`.
   `SM_Typo` appears nowhere in the response.
2. **Partial batch.** Set the read-only attribute on one of two scratch `.uasset` files, then
   `asset.bulk_delete` both. Expect `success: true` and `deleted[]` naming both, while the
   read-only asset is still in the registry and still on disk.

## Impact

High. Silent false-success on the verb's normal path: the caller is told assets are gone and
builds on it — re-creating an asset at a path that is still occupied, reporting a cleanup as
complete, or moving on from content that still ships. The response contains no field that can
contradict the claim, so there is no client-side check short of a follow-up `asset.exists` per
path, which is the work the verb exists to batch.

Severity rationale: impact = silent false-success on a normal path (High band) × reach = the
documented preferred verb for any multi-asset cleanup, though rarer per-session than
`asset.delete` → stays **High**, not bumped.

**Workaround:** delete through `asset.delete` (per-entry `existsAfter` / `existsOnDisk`
/ `inMemoryReferencers`) when it matters that the files are gone, or re-probe every path with
`asset.exists` after the bulk call. Both are already recommended in `Docs/wiki-src/asset.md`.

## Fix

Apply the pattern `asset.delete` now uses in `Handlers/Asset/AssetManageHandler.cpp` — the
engine's return value is recorded as a *claim* and reconciled against a probe, never emitted
as the outcome:

- Classify `existedBefore` per path **before** mutating (a typo'd path must not probe as
  "gone" afterwards and count as a deletion).
- After `ObjectTools::DeleteObjects`, re-probe each path against **both** the asset registry
  (`AssetUtils::VerifyAssetExists`) and the file on disk
  (`AssetUtils::DoesPackageFileExistOnDisk`, exported for this in `Utils/AssetUtils.h`);
  `existsAfter` is the union, and `existsOnDisk` is emitted separately so the memory/disk
  divergence case is visible.
- Emit a per-entry `results[]` with `path` / `existedBefore` / `existsAfter` / `existsOnDisk`
  / `deleted` / `missing`, plus top-level `deleted[]` / `failed[]` / `missing[]` built from
  the probe, not from `ValidPaths`. Include the paths dropped by the `DoesAssetExist` /
  `LoadAsset` filter as `missing`, and make `requestedCount` the caller's array length.
- `success` = nothing survived (`!bAnySurvived`), not `DeletedCount > 0`. Partial batch
  failure is failure of the request that was made.
- On failed entries, name the holders via the same `GatherInMemoryPackageReferencers` helper
  (`AssetManageHandler.cpp`) and flag `memoryDiskDivergence` when the count claimed a delete
  but the file is still there.

`DeletedCount` should be kept in the response as an engine-reported figure only, clearly
distinct from the probed `deletedCount`, since `AddExtraObjectsToDelete` means it can exceed
the request.

**No existing consumer blocks this.** `Tests/Assets/TestBulkDeleteRedirectorScope.cpp` is the
only test dispatching this verb and reads only `redirectorsDeletedOutsideScope`; nothing else
in the plugin reads `success` / `deleted` / `requested` from it.

## History

- `#1-source-review-no-verification` `OPEN` reporter — Found while fixing the sibling
  `B-asset-delete-force-delete-leaves-uasset-on-disk`, whose IN-REVIEW note deferred this verb
  to its own ticket. Verified against current source, not observed live: the handler's
  `deleted[]` / `success` / `requested` construction in `AssetWorkflowHandler.cpp`, and the
  engine chain `ObjectTools::DeleteObjects` → `DeleteItems` → `FAssetDeleteModel::DoDelete` →
  `DeleteObjectsUnchecked` → `CleanupAfterSuccessfulDelete` in UE 5.8 source. Contradicts the
  lead on one point, recorded above: the bulk path passes `bPerformReferenceCheck=false`, so
  the referencer cull does not fire here — the silent losses come from
  `MakeReadOnlyPackageWritable` / `DeleteSingleObject` skips, the `DoesPackageExist` cull, the
  unchecked `IFileManager::Delete`, and `AddExtraObjectsToDelete` inflating the count.
- `#2-reconciled-against-a-probe` `IN-REVIEW` developer — `AssetWorkflowHandler.cpp`
  `asset.bulk_delete` now records one `FRequestedDelete` per path the CALLER sent
  (`path` / `existedBefore` sampled before the delete / `attempted`), keeps
  `ObjectTools::DeleteObjects`' return only as `engineDeletedCount`, and after the call
  re-probes every requested path against `UEditorAssetLibrary::DoesAssetExist` **union**
  `AssetUtils::DoesPackageFileExistOnDisk`. Response is now `results[]`
  (`path`/`existedBefore`/`attempted`/`existsAfter`/`existsOnDisk`/`deleted`/`missing`, plus
  `memoryDiskDivergence` when the registry row is gone and the file is not) with
  `deleted[]`/`failed[]`/`missing[]`, `deletedCount`/`failedCount`/`missingCount`/
  `attemptedCount`, `existsAfter`, and a `failureHint`; `success` is `!bAnySurvived` and the
  envelope now errors `BULK_DELETE_FAILED` (carrying the body) on a partial batch instead of
  reporting it as a success. `requested` (the count that LOADED) is replaced by
  `requestedCount` (the caller's array length), which closes the second defect: dropped paths
  now appear as `missing` with `attempted:false` rather than vanishing. The sibling's
  referencer logic was deliberately NOT imported — this route passes
  `bPerformReferenceCheck=false` and is gated by `FAssetDeleteModel::CanDelete()`, confirmed
  in UE 5.8 source, so the cull does not fire here. Regression coverage:
  `Tests/Assets/TestBulkDeleteVerification.cpp` —
  `PinWright.asset.bulk_delete.PartialBatchIsReportedAsPartial` forces a deterministic partial
  batch with a one-object `FEditorDelegates::OnAssetsCanDelete` veto (the route
  `DeleteObjectsUnchecked` → `DeleteSingleObject` takes) and asserts the survivor is in
  `failed[]` not `deleted[]` while `engineDeletedCount > 0`; `.UnattemptedPathIsReportedAsMissing`
  asserts a never-created path reaches `missing[]` with `requestedCount == 2`. Both fail on the
  old shape. `Docs/wiki-src/asset.md` § asset.bulk_delete rewritten (it documented the defect).
  NOT COMPILED OR RUN (instructed): a suite pass must confirm the two new tests and, in
  particular, that `PinWright.asset.bulk_delete.OutsideRedirectorsSurviveTheDefaultScope` stays
  green — its `Capture.bSuccess` assertion now depends on the deleted in-memory fixture actually
  probing absent after `CleanupAfterSuccessfulDelete`'s `CollectGarbage`.
- `#3-first-field-hit-of-the-whole-batch-abort` `IN-REVIEW` reporter — **Second encounter, from the field. Status deliberately unchanged: this is neither a verification of `#2` nor a return.** The lead that produced it was *"`asset.bulk_delete` is all-or-nothing and reports false failures"*; **the "false failures" half did not survive re-derivation and is corrected here**, and what is left is a real residue this ticket should carry rather than a new ticket.

  **Measured, this session's Procedural Vegetation pass.** Four scratch PCG graphs under `/Game/PinWrightScratch/PVTest/` submitted to `asset.bulk_delete` in one call: `BULK_DELETE_FAILED`, `engineDeletedCount: 0`, and all four entries `attempted: true, deleted: false`. With no intervening change, singular `asset.delete` on three of those exact paths succeeded immediately; only the fourth was genuinely referenced, and `asset.delete` named its holder precisely — `…StaticMeshActor_48.PCGComponent_0.PCGGraphInstance`, `referencedByUndoBuffer: true` — and refused with no side effects. (The four assets no longer exist, so that comparison is not re-derivable at HEAD; everything below it is re-derived from source.)

  **Correction 1 — the receipt did not report false failures. It was telling the truth.** `#2`'s per-path probe works exactly as `#2` says it does: the accounting loop at `AssetWorkflowHandler.cpp:651-698` sets each entry's `deleted` to `Requested.bExistedBefore && bGone` (`:671`), where `bGone` is a *fresh* two-probe readback taken after the engine call — `UEditorAssetLibrary::DoesAssetExist` at `:657` and an on-disk package probe at `:658` — and `EngineDeletedCount` is never consulted in that loop. All four rows read `deleted: false` because at that moment all four still existed. So the receipt is accurate and `#2`'s fix is doing its job on this path; the lead's framing blamed the one part of the handler that behaved.

  **Correction 2 — the engine call is `ObjectTools::DeleteObjects`, not `DeleteObjectsUnchecked`.** `AssetWorkflowHandler.cpp:581-582`. `DeleteObjectsUnchecked` occurs in this file only inside comments (`:572`, describing the call chain, and `:724`, in the `failureHint` text), so citing it as the call site would send a fixer to the wrong frame.

  **What is actually wrong, and it is `#2`'s own predicted case arriving in the field.** `#2`'s body already names the mechanism: `FAssetDeleteModel::CanDelete()` is `!CanForceDelete()`, so a single referenced member makes `DoDelete()` return false for the **whole batch**, `ObjectsDeleted` stays 0, and the handler correctly errors — a case `#2` labelled *"honest by accident"*. This is the first measurement of it, and it shows the label is too generous by exactly one level. The receipt is honest **per row** and silent **about the batch**: four identical `deleted: false` rows, `engineDeletedCount: 0`, and nothing anywhere in the payload distinguishing *"the batch was refused as a unit because of one member"* from *"each of these four failed on its own merits"*. Those two states demand opposite next actions — re-submit the other three, versus investigate all four — and the response cannot tell them apart. The caller's only route to the truth is to abandon the batch verb and re-issue N singular `asset.delete` calls, which is what this session did, and which is the fallback the rubric's Medium band describes.

  **The fix is already written, in this plugin, one file away.** `asset.delete` produces precisely the missing attribution: `GatherInMemoryPackageReferencers` (`Handlers/Asset/AssetManageHandler.cpp:114-146`) calls `ObjectTools::GatherObjectReferencersForDeletion` at `:128-129` and harvests referencer object paths at `:135` and `:142`, and the refusal branch publishes them per entry at `:773-778` as `refused` / `errorCode: ASSET_IN_USE` / `refusalReason` / `referencers`, plus `referencedByUndoBuffer` at `:762`. `asset.bulk_delete`'s failure branch could run the same gather over the surviving paths and fill in the same fields — turning four indistinguishable rows into one blocker and three re-submittable paths. **Ask: on the `EngineDeletedCount == 0 && every member survived` branch, say so explicitly (a `batchRefused` bool or equivalent) and attribute it**, reusing `asset.delete`'s gather rather than a second implementation.

  **Contrast worth recording for whoever picks this up: singular `asset.delete` is the template, not a rival.** It named the exact holder including the sub-object chain, reported `referencedByUndoBuffer`, refused, and left nothing behind. Nothing needs designing here; the batch verb needs to call what the singular verb already calls.

  **Also confirmed while measuring, and it is good design, not a gap:** `success` is `!bAnySurvived` (`:703`) and the whole thing is sent as `BULK_DELETE_FAILED` (`:772`), but `Result` travels **with** the error (`:776`), so the per-entry truth is readable on the failure path. The gap is attribution, not transport.

  **Why no new ticket, and why not DONE.** Same verb, same handler, same response object, and `#2` already names the whole-batch abort in its body — a separate ticket would put two authorities on one receipt, which is what the board's dedup and no-umbrella rules exist to prevent. And this is explicitly **not** a verification of this ticket's own subject: the title is about a *partially*-deleted batch reporting as a clean success, and nothing was deleted in this measurement, so the partial-batch path was never exercised. A tester still has to drive a genuine partial batch before this can close. **Severity left at High, unchanged and not re-argued** — the residue recorded here is narrower than the defect the ticket was filed for (a missing attribution field on a failure branch that already reports honestly would rate Medium on its own, as *"a readback omits a field and forces a fallback"*), and an encounter should not deflate a ticket whose original subject is untouched. `encounters` 1 → 2 per the dedup-append rule; that is a work-ordering tiebreak, never a severity input.
