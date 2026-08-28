---
id: B-force-delete-nulls-referencers
title: "asset.delete force-deletes unconditionally: every in-memory pointer to the target is nulled and its owner marked dirty BEFORE the engine decides whether the package may go, so a declined delete leaves the .uasset alive and its referencers broken"
status: IN-REVIEW
severity: Critical
category: bug
tags: [asset, asset-delete, level-delete, animation-cleanup, force-delete, objecttools, data-loss, destructive-default, no-preflight, referencers, engine-behaviour]
encounters: 1
lastSeen: 2026-08-28T00:00:00Z
---

# `asset.delete` performs Force Delete with no opt-in, no pre-flight, and no rollback

Sibling ticket `B-asset-delete-force-delete-leaves-uasset-on-disk` (IN-REVIEW) fixed the *reporting*
half of this: `asset.delete` now reconciles the engine's return against `existsAfter` / `existsOnDisk`,
names `inMemoryReferencers`, and flags `memoryDiskDivergence`. That fix is correct and is not
disputed here. **This ticket is the destructive half, which the reporting fix does not touch:** by
the time the plugin computes those referencers, the engine has already nulled every pointer to the
doomed asset editor-wide and marked each referencing package dirty. The report is a post-mortem.

## Mechanism (confirmed in UE 5.8 engine source, not inferred)

`C:\UE_5.8\Engine\Source\Editor\UnrealEd\Private\ObjectTools.cpp`, `ObjectTools::ForceDeleteObjects`,
in execution order:

1. `RecursiveRetrieveReferencers` → close asset editors for every referencer. **This is the only
   early-out before anything destructive:** if an editor refuses to close, `return 0`.
2. Logs `Force Deleting N Package(s)` (`LogUObjectGlobals`).
3. Destroys BP instance actors/components and SCS nodes; reparents + recompiles child Blueprints;
   `RemoveChildRedirectors` / `RemoveGeneratedClasses`.
4. **`ForceReplaceReferences(nullptr, ObjectsToReplace, ReplaceInfo, false)`.** This is the
   destructive step. Because `ForceDeleteObjects` passes an **empty** `ObjectsToReplaceWithin`,
   `ObjectTools::ForceReplaceReferences` takes its `FThreadSafeObjectIterator` branch and walks
   **every live `UObject` in the editor**, running `FArchiveReplaceObjectAndStructPropertyRef` with a
   replacement map of `{doomed object → nullptr}`. For each object it touched it then calls
   `PostEditChangeProperty(EPropertyChangeType::Redirected)` and **`CurReplaceObj->MarkPackageDirty()`**.
   It also strips `RF_RootSet` from the doomed objects and never restores it.
5. `CollectGarbage`, `OnAssetsPreDelete`.
6. `DeleteSingleObject` per object. **This is what the return value counts.**
7. **`CleanupAfterSuccessfulDelete(PotentialPackagesToDelete)`** — default `bPerformReferenceCheck = true`
   (`Editor\UnrealEd\Public\ObjectTools.h`). Per package it runs
   `ObjectTools::GatherObjectReferencersForDeletion` and, `if (bIsReferenced)`, executes
   `PackagesToDelete.RemoveAt(PackageIdx)` — a **silent cull**. A culled package gets no
   `SetDirtyFlag(false)`, no `FAssetRegistryModule::PackageDeleted`, no `UnloadPackages`, and no file
   delete. No log line, no warning, no return-value adjustment.
8. Returns `NumDeletedObjects` from step 6, **never reconciled against step 7**.

Three facts make this a loss rather than a race:

- **Nothing restores the nulled references.** `ForceDeleteObjects` contains no `FScopedTransaction`;
  `FArchiveReplaceObjectAndStructPropertyRef` issues no `Modify()`, so the nulling is not recorded in
  the undo buffer and cannot be undone. Worse, `CleanupAfterSuccessfulDelete` calls
  `GEditor->ResetTransaction(...)` when a package is undo-referenced — actively destroying whatever
  undo history did exist.
- **The caller is never told which objects were damaged.** `ForceReplaceReferences` populates
  `FForceReplaceInfo::DirtiedPackages` / `ReplaceableObjects`, but `ForceDeleteObjects` declares
  `FForceReplaceInfo ReplaceInfo;` as a scoped local and discards it.
- **Epic documents the mismatch in-source.** The terminal `ensureMsgf` in `ForceDeleteObjects` reads:
  *"To fix this ensure you need to fix the reference replacement/isreferenced mismatch. The former
  does not find subobjects, the latter does."* That mismatch is precisely the window in which step 4
  nulls and step 7 declines. (That `ensure` only covers step-6 failures; a step-7 cull fires nothing.)

## Reachability — force is not opt-in, it is the only delete available

`UEditorAssetLibrary::DeleteAsset` / `DeleteDirectory` / `DeleteLoadedAsset(s)` all route through
`UEditorAssetSubsystem` (`Editor\UnrealEd\Private\Subsystems\EditorAssetSubsystem.cpp`) and **every
one of them calls `ObjectTools::ForceDeleteObjects` with `bShowConfirmation = false`**. There is no
safe-delete entry point in `UEditorAssetLibrary` at all. So a plugin verb built on it force-deletes
whether or not anyone asked.

| Verb | Handler (symbol) | Reaches | `force` param? | Pre-flight refusal? |
|---|---|---|---|---|
| `asset.delete` | `AssetManageHandler.cpp`, the `asset.delete` handler body | `UEditorAssetLibrary::DeleteAsset` / `DeleteDirectory` → `ForceDeleteObjects` | **none** | **none** |
| `level.delete` | `LevelHandler.cpp`, the `level.delete` handler body | same | **none** | **none** — and its own registered summary claims *"Errors if the level is currently loaded or referenced by other assets"*, a guard that does not exist in the code |
| `animation.cleanup` | `AnimationHandler.cpp`, the `animation.cleanup` handler body | same | **none** | **none** — it runs `CloseAllEditorsForAsset` + `FlushRenderingCommands` + `ForceGarbageCollection(true)` first, i.e. it deliberately strengthens the force delete |
| `blueprint.create_enum` | `BlueprintTypeDefinitionHandler.cpp`, the rollback path | same | n/a | n/a — deletes only the asset it just created |
| `asset.bulk_delete` | `AssetWorkflowHandler.cpp`, the `asset.bulk_delete` handler body | `ObjectTools::DeleteObjects` → `FAssetDeleteModel::DoDelete` → `DeleteObjectsUnchecked` | n/a | **implicitly safe** — see below |

The asymmetry is worth stating plainly: **the batch verb is safe and the single verb is not.**
`ObjectTools::DeleteObjects` reaches `FAssetDeleteModel`, whose `CanDelete()` is literally
`!CanForceDelete()`; when anything references the target it returns false, `DeleteItems` logs
*"Could not delete"*, returns 0, and **`ForceReplaceReferences` is never reached**. The engine's own
model treats referenced-delete as a separate, gated operation. `asset.delete` bypasses that gate.

What `asset.delete` does today, in order (`AssetManageHandler.cpp`, the `asset.delete` handler body):

- **Before** the delete: `GetReferencingPackages` (registry `GetReferencers`, on-disk only) —
  accumulated into `AggregatedReferencers` **for reporting**. Its own comment says why it runs early:
  *"Query referencers BEFORE deleting — after deletion the registry would have already purged those
  entries."* No branch, no error code, no refusal.
- The delete: `UEditorAssetLibrary::DeleteAsset` — unconditional once `bExistedBefore`.
- **After** the delete, and only on `if (bExistedBefore && !bGone)`: `GatherInMemoryPackageReferencers`
  → `ObjectTools::GatherObjectReferencersForDeletion`. This is the walk that could have predicted a
  broken reference, and it runs strictly after the references are gone.

So the plugin already imports the exact engine predicate that would gate this — it just calls it too
late and only on the failure path.

## Repro sketch (not executed — see below)

1. Pick an asset `A` referenced by asset `B` (`asset.dependencies` on `A` → `referencerCount > 0`).
2. Arrange for `A`'s package to remain referenced in memory at cull time. Any holder the engine's
   `GatherObjectReferencersForDeletion` finds but `ForceReplaceReferences` misses will do — the
   sibling ticket names the transaction buffer and PinWright's own cached Niagara system view models
   in `FPluginState` as candidates, and the engine's own `ensureMsgf` names the general class
   (subobject references).
3. `asset.delete { path: "/Game/.../A" }`.
4. Expected observation: the response reports `existsAfter: true`, `memoryDiskDivergence: true`
   (i.e. the delete failed) — **and** `B`'s reference to `A` is already null and `B`'s package is
   already dirty. `editor.list_dirty_packages` should show `B`.
5. Any subsequent save writes the null: `editor.save_all` (every dirty world + content package,
   no filter), `editor.quit { save: true }` (`FEditorFileUtils::SaveDirtyPackages(bPromptUserToSave=false,
   bSaveMapPackages=true, bSaveContentPackages=true)` — silent), or `asset.save { assetPath: B }`.
   Worse, `asset.fixup_redirectors` (and `asset.bulk_delete`'s default fixup) reaches
   `RedirectorFixupPolicy::FixupReferencers`, which calls
   `FEditorFileUtils::PromptForCheckoutAndSave(ReferencingPackages, /*bCheckDirty=*/false,
   /*bPromptToSave=*/false, ...)` — `bCheckDirty=false` means it writes referencing packages that
   were not even dirty. Running a redirector fixup after a delete batch is the documented workflow.

**Not executed, deliberately.** Reproducing this means deliberately corrupting a referencing asset in
a shared editor that other agents are working in, and the destructive step is irreversible (no
transaction, and `ResetTransaction` may already have cleared the buffer). The mechanism above is read
from engine source end to end; the *conjunction* (referenced target + cull declines + a later save)
has not been observed in one run.

## Impact

**Critical** by the README rubric: impact class = *"a write that corrupts or loses asset data"*;
reach = `asset.delete` is a routine cleanup verb and force is unconditional, so **every** delete of a
referenced asset takes this path — not a rare edge. No reach bump-down.

Being precise about what is and is not demonstrated, so a reader can disagree with the rating on the
facts rather than on the framing:

- **Unconditional and certain:** deleting a referenced asset through `asset.delete` / `level.delete` /
  `animation.cleanup` nulls every in-memory pointer to it editor-wide and marks each referencing
  package dirty, with no undo and no report of which packages were touched. That is true on the
  success path too — it is what Force Delete means — except that through the plugin nobody opted in
  to Force Delete, and interactive UE never does this without showing the referencer list behind a
  separate red-button confirm.
- **Certain, and the reason this is not merely "force delete working as designed":** when
  `CleanupAfterSuccessfulDelete` culls the package, the destruction is *unpaired*. The `.uasset`
  survives with all its data; only the things that pointed at it are destroyed. Nothing in the engine
  or the plugin reverses that.
- **Derived from source, not observed:** the step from in-memory nulls to bytes on disk. It needs one
  save of a package the engine itself marked dirty. Five in-plugin verbs do that, two of them
  (`editor.quit {save:true}`, `asset.fixup_redirectors`) without the caller naming the package. UE's
  own autosave does **not** overwrite originals (it writes to `Saved/Autosaves/`), so this is not a
  no-user-action-at-all path — but it is well short of requiring a deliberate choice to re-save.

The sibling's `failureHint` now tells the caller not to re-save the referencing assets. That is the
right advice and it is not sufficient: in a shared editor another agent saves on its own schedule,
and `editor.save_all` / `editor.quit` do not consult this caller.

**Workaround:** none inside the plugin for the destructive half. `asset.dependencies` before
`asset.delete` narrows exposure (skip the delete when `referencerCount > 0`), but that is the
on-disk registry question and misses the in-memory holders that actually drive the cull. After a
delete that reports `memoryDiskDivergence`, treat every package in `referencingBlueprints` as
suspect, do not save anything, and `asset.reload` those packages to discard the nulled pointers
before any save verb runs — `asset.reload` re-reads from disk and is the only in-plugin undo for this.

**Fix:** the honest answer is **do not call the engine's force path by default**. Concretely, in the
`asset.delete` handler (and the same for `level.delete` / `animation.cleanup`), before any delete call:

1. **Pre-flight with the engine's own predicate.** `ObjectTools::GatherObjectReferencersForDeletion`
   is `UNREALED_API` (`Editor\UnrealEd\Public\ObjectTools.h`) and the plugin already wraps it as
   `GatherInMemoryPackageReferencers` in `AssetManageHandler.cpp` — move that call **above** the
   delete instead of onto the failure path. Union it with the existing registry `GetReferencingPackages`.
2. **Route on the result, mirroring `FAssetDeleteModel`'s own split.** Clean → `ObjectTools::DeleteObjectsUnchecked`
   (also `UNREALED_API`; it never calls `ForceReplaceReferences`, still runs `AddExtraObjectsToDelete`,
   and passes `bPerformReferenceCheck=false` to both `DeleteSingleObject` and
   `CleanupAfterSuccessfulDelete` — safe precisely because the plugin established unreferencedness
   first). Referenced → refuse with `ERR_ASSET_IN_USE` and the referencer list, unless the caller
   passes a new explicit `force: true`.
3. **Make `force: true` state its price** in the param description and in the response: it nulls every
   in-memory reference irreversibly, and the file may still survive. `AssetCreatePolicy::Resolve`
   already implements exactly this refusal shape (`ReferencerCount > 0` → `ERR_ASSET_IN_USE`) for the
   `overwrite:true` create verbs — copy it rather than inventing a second idiom.
4. **On the `force` path, report the collateral.** `ForceDeleteObjects` discards its `FForceReplaceInfo`,
   so `DirtiedPackages` is unreachable — snapshot `UPackage::IsDirty()` across the pre-flight
   referencer set before and after the call and return the newly-dirtied packages as
   `referencesNulled` / `doNotSave`.

**Tradeoffs, stated rather than hidden:**

- **This is a breaking behaviour change.** Any caller relying on `asset.delete` silently removing a
  referenced asset now gets `ERR_ASSET_IN_USE` and must pass `force: true`. That is the point, but it
  will surface as "asset.delete stopped working" in existing scripts and skills.
- **Cost on the normal path.** `GatherObjectReferencersForDeletion` walks every live `UObject`
  (`FReferencerFinder::GetAllReferencers`). `ForceDeleteObjects` already pays this twice (in
  `DeleteSingleObject` and in `CleanupAfterSuccessfulDelete`), so a pre-flight adds a third walk on
  the refuse path and a *first* one on the `DeleteObjectsUnchecked` path, which previously paid none.
  On a large loaded editor that is a real per-delete cost.
- **It does not make `force: true` safe.** The engine's reference-replacement/`IsReferenced` mismatch
  is inside `ObjectTools` and cannot be fixed from the plugin. With `force`, references can still be
  nulled while the package survives. The plugin can only make that case opt-in, named, and reported
  (item 4) — never absent.
- **The gate can be wrong in the safe direction.** If the plugin's pre-flight finds a referencer the
  engine's cull would not have, the delete is refused where it would have succeeded. The cost is one
  `force: true`; the inverse error costs an asset's references. Prefer the false refusal.
- **`DeleteObjectsUnchecked` skips the editor-closing step** that `ForceDeleteObjects` performs, so an
  open asset editor on the target would be left holding a torn-down object. Pre-flight must therefore
  also refuse (or close, via `editor.close_asset`) when an editor is open — `animation.cleanup` already
  calls `CloseAllEditorsForAsset` and shows the shape.

Rejected: **a transacted approach**. `ForceDeleteObjects` has no `FScopedTransaction`, and
`FArchiveReplaceObjectAndStructPropertyRef` calls no `Modify()`, so a plugin-side transaction wrapping
the call would record nothing to roll back; and `CleanupAfterSuccessfulDelete` can call
`GEditor->ResetTransaction(...)` mid-operation, which would discard the plugin's own transaction along
with everything else. This is not a judgement call — it is read from the source.

## History

- `#1-engine-ordering-confirmed` `OPEN` reporter — UE 5.8 (`C:\UE_5.8\Engine\Source`). Surfaced while
  fixing `B-asset-delete-force-delete-leaves-uasset-on-disk`, which left this half open by design (see
  its `## Impact`, *"A fixer should consider re-rating this Critical"*). Engine ordering verified by
  reading `ObjectTools::ForceDeleteObjects` end to end: `ForceReplaceReferences(nullptr, ...)` runs at
  step 4, `DeleteSingleObject` (the return value's source) at step 6, `CleanupAfterSuccessfulDelete`'s
  `if (bIsReferenced) PackagesToDelete.RemoveAt(...)` cull at step 7 — confirmed in source, not
  inferred from logs. No restore path exists: no `FScopedTransaction` in the function, no `Modify()`
  in the replace archive, `FForceReplaceInfo` discarded as a scoped local, and
  `GEditor->ResetTransaction` called in the cull. Reachability established by reading
  `UEditorAssetSubsystem::DeleteAsset` / `DeleteLoadedAsset` / `DeleteLoadedAssets` / `DeleteDirectory`
  — all four call `ForceDeleteObjects` unconditionally with `bShowConfirmation=false`, so no plugin
  verb built on `UEditorAssetLibrary` can avoid the force path; `asset.delete`, `level.delete`,
  `animation.cleanup` and the `blueprint.create_enum` rollback all do. `asset.bulk_delete` does **not**
  (it uses `ObjectTools::DeleteObjects`, which gates on `FAssetDeleteModel::CanDelete() == !CanForceDelete()`).
  Current `asset.delete` pre-flight confirmed to be report-only: `GetReferencingPackages` before the
  delete feeds `referencingBlueprints`; `GatherInMemoryPackageReferencers` runs after it and only on
  the failure branch. **Not reproduced by deleting anything** — the mechanism is source-derived and
  the in-memory nulling is certain, but the on-disk persistence step (a save of a package the engine
  marked dirty) was not exercised. Rated Critical on impact class *"a write that corrupts or loses
  asset data"* x reach *routine verb, force unconditional*; downgrade to High is defensible if a
  reviewer holds that unobserved on-disk persistence keeps it out of the Critical band.
- `#2-gate-behind-force-opt-in` `IN-REVIEW` developer — "Gated all three verbs behind a new
  `force` parameter (default false) routed through one new shared helper
  `Source/PinWright/Private/Utils/AssetDeletePolicy.{h,cpp}` (`DeleteAsset` / `DeleteDirectory`).
  **Deviated from the Fix section on the mechanism, deliberately.** The ticket proposes hoisting
  the plugin's `GatherInMemoryPackageReferencers` above the delete and routing clean →
  `DeleteObjectsUnchecked`. That pre-flight is wrong as written and would have refused every
  delete: it gathers over the **package**, and `GatherObjectReferencersForDeletion` sets
  `bOutIsReferenced` from any object inside `InObject` carrying `GARBAGE_COLLECTION_KEEPFLAGS`
  (= `RF_Standalone` in the editor, `GarbageCollection.h:28`) — pre-delete that is the asset
  itself. The existing wrapper is only meaningful *after* `DeleteSingleObject` clears
  `RF_Standalone`, which is exactly why it sits on the failure path. The clean path instead calls
  `ObjectTools::DeleteObjects(Objects, bShowConfirmation=false, CancelNotAllowed)`, the engine's own
  safe delete: `DeleteItems` builds an `FAssetDeleteModel`, `DoDelete()` early-outs on
  `CanDelete() == !CanForceDelete()` having mutated nothing, and otherwise reaches the very
  `DeleteObjectsUnchecked` the ticket names. That is the same primitive `asset.bulk_delete` already
  uses, so the two verbs no longer disagree; it duplicates no predicate, so the ticket's *cost on the
  normal path* and *gate can be wrong in the safe direction* tradeoffs both disappear — the refusal
  is the engine's own decision and the referencer walks run only after one. **`force:true` survives**
  (opt-in, priced in its param description and in the response): `ForceDeleteObjects` is the only way
  to delete a referenced asset, interactive UE offers exactly that behind a red-button confirm, and
  removing it would leave callers unable to do something the editor can. Ticket item 4 implemented:
  `ForceDeleteObjects` discards its `FForceReplaceInfo`, so the policy brackets the force call with a
  `TObjectIterator<UPackage>` dirty-set snapshot and returns the newly-dirtied packages as
  `referencesNulled`, with a hint naming every verb that would write the nulls and pointing at
  `asset.reload`. **Two ticket claims corrected.** (a) The tradeoff *'`DeleteObjectsUnchecked` skips
  the editor-closing step'* is wrong: it calls `DeleteSingleObject`, which calls
  `CloseAllEditorsForAsset` unconditionally before its reference check (`ObjectTools.cpp:3470`); what
  it skips is only `ForceDeleteObjects`' recursive close of *referencers'* editors, which cannot
  matter on a path that refuses when referencers exist. No editor-open pre-flight was added. (b) A
  new hazard the ticket does not name: the safe path's terminal `CleanupAfterSuccessfulDelete` runs
  with `bPerformReferenceCheck=false`, so for the rare package holding **two** `RF_Standalone`
  assets, deleting one now removes the shared `.uasset` (taking the other with it) where the force
  path's cull used to leave the file alone. That is engine-standard Content Browser behaviour and it
  keeps `PinWright.asset.delete.VerdictMatchesExistsAfter` green (its assertions are relational), but
  it is a real edge-case change. **Files:** new `Utils/AssetDeletePolicy.{h,cpp}`;
  `Handlers/Asset/AssetManageHandler.cpp` (`asset.delete`: `force` param, policy routing, per-entry
  `refused` / `errorCode` / `refusalReason` / `referencesPreserved` / `referencers`, per-entry and
  top-level `forced` / `referencesNulled`, top-level `errorCode`, extended `failureHint`; the
  'may now contain broken references' warning is now emitted only when something was actually
  deleted); `Handlers/Level/LevelHandler.cpp` (`level.delete`: `force` param, `ASSET_IN_USE`
  refusal — its registered summary has always claimed this guard and now has it; raw `TEXT()` code
  because that file does not adopt `ErrorCodes::`); `Handlers/Animation/AnimationHandler.cpp`
  (`animation.cleanup`: `force` param, new `refused[]` bucket kept out of `failed[]`,
  `referencesNulled`, `CLEANUP_PARTIAL` now counts refusals). `blueprint.create_enum`'s rollback was
  left forcing — it deletes only the asset it just created. Docs: `Docs/wiki-src/asset.md`
  `### asset.delete` and `Docs/wiki-src/level-building.build-scripts.md` (whose 'their references are
  nulled' advice was stale). **Regression test:**
  `Source/PinWright/Private/Tests/Assets/TestAssetDeleteForceGate.cpp`,
  `PinWright.asset.delete.RefusesReferencedInsteadOfNullingReferences`. It judges on the side effect,
  not the wire text: using `AssetRefDirectionFixtures::BuildHardDependency` (A references B), it
  asserts that after `asset.delete{path:B}` the asset survives **and A's package is still clean** —
  before the fix `ForceReplaceReferences` calls `MarkPackageDirty()` on every object it rewrites
  (`ObjectTools.cpp:1470`) and B is deleted outright, so both halves fail. Case 2 asserts the
  unreferenced path still deletes (A, which nothing references). **Breaking change, stated plainly:**
  any caller relying on `asset.delete` / `level.delete` / `animation.cleanup` silently removing a
  referenced asset now gets `ASSET_IN_USE`; a folder is gated as one batch, so one externally-held
  asset refuses the whole folder. Not compiled and not run — the orchestrator owns builds."
