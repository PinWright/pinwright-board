---
id: B-asset-import-overwrite-commit-kills-editor
title: "asset.import over an existing asset kills the editor: FScopedOverwriteStage::Commit calls ObjectTools::ForceReplaceReferences, which walks every UFunction's bytecode and faults with EXCEPTION_ACCESS_VIOLATION reading 0xffffffffffffffff"
status: IN-REVIEW
severity: Critical
category: bug
tags: [asset, import, overwrite, objecttools, ForceReplaceReferences, crash, editor-killer, texture, multi-agent, data-loss]
encounters: 2
lastSeen: 2026-09-05T20:02:02Z
---

# `asset.import` over an existing asset faults inside `ForceReplaceReferences` and takes the editor down

## Symptom

A re-import of an existing texture killed a shared editor outright at
`2026-09-05T20:02:02Z`, six seconds after the Interchange import itself had already reported
success. Every agent in that editor lost its session.

```
[20.01.56:223][779] LogPinWrightSafePoint: Deferring work by one safe-point core-ticker hop (asset.import).
[20.01.56:243][779] LogInterchangeEngine: Display: Interchange start importing source
                    [X:/src/unreal/EAContentExamples58/Docs/fps/data/textures/T_WPN_Markings_M.png]
[20.01.56:247][779] LogTexture: Display: Building texture TwoD:
                    /Game/FPS/Weapons/Textures/T_WPN_Markings_M (TFO_AutoDXT, 2048x512)
[20.01.56:258][779] LogInterchangeEngine: Display: Interchange import completed [...]
[20.01.56:368][779] LogDatasmithContent: Do not use the UDatasmithStaticMeshCADImportData ...
[20.02.02:993][779] LogWindows: Error: === Critical error: ===
[20.02.02:993][779] LogWindows: Error: Unhandled Exception: EXCEPTION_ACCESS_VIOLATION
                    reading address 0xffffffffffffffff
```

Callstack, PinWright frames in bold order — the fault is reached from PinWright's own overwrite
path, not from an engine-initiated import:

```
FPropertyProxyArchive::operator<<()            PropertyProxyArchive.h:46
UStruct::SerializeExpr()                       ScriptSerialization.inl:243
UStruct::SerializeExpr()                       Class.cpp:2691
UStruct::Serialize()                           Class.cpp:2458
UFunction::Serialize()                         Class.cpp:7608
FFindReferencersArchive::ResetPotentialReferencer()  FindReferencersArchive.cpp:89
ObjectTools::ForceReplaceReferences'::<lambda_1>::operator()()  ObjectTools.cpp:1314
ObjectTools::ForceReplaceReferences()          ObjectTools.cpp:1369
ObjectTools::ForceReplaceReferences()          ObjectTools.cpp:1497
ObjectTools::ForceReplaceReferences()          ObjectTools.cpp:1504
AssetImportPolicy::FScopedOverwriteStage::Commit()   AssetImportPolicy.cpp:1010
AutoHandler_314_'::<lambda_1>::operator()()    AssetManageHandler.cpp:502
PinWrightSafePoint::DeferRequestToSafePoint    SafePoint.h:474
FRpcDispatcher::DeferActiveRequestToSafePoint  RpcDispatcher.cpp:1078
FEngineLoop::Tick()
```

Crash report: `Saved/Crashes/UECC-Windows-01FD4B9D466127197DF55DB7729358DA_0002`
(23:02:01 local = 20:02:01 UTC). Two earlier reports exist under the same session id
(`_0000` 21:52:19, `_0001` 22:16:33 local) — this editor had already died twice today.

## Mechanism

`ForceReplaceReferences` takes its `FThreadSafeObjectIterator` branch and calls
`FFindReferencersArchive::ResetPotentialReferencer` on **every live UObject**, which for a
`UFunction` serialises its compiled bytecode through `FPropertyProxyArchive`. `0xffffffffffffffff`
is not a null deref — it is a poisoned or stale `FProperty*` inside a script expression, so the
walk is reading a `UFunction` whose bytecode holds a dangling property pointer. Any Blueprint
recompiled earlier in the session can leave one behind: this editor had run
`blueprint.compile` on `/Game/FPS/Weapons/BP_Weapon_AR` at `20:01:45` and two
`blueprint.set_default` calls at `20:01:06` / `20:01:10`, eleven seconds before the fault.

**The blast radius is the whole process, and it is not the caller's asset.** The import had
already succeeded; what died was the reference fix-up afterwards. Nothing the importing caller
passed can make this safe, and nothing any *other* agent in the editor did can protect it —
three other streams were mid-work in this editor and all lost their sessions.

`B-asset-delete-force-delete-leaves-uasset-on-disk` / `B-force-delete-nulls-referencers` already
document `ObjectTools::ForceReplaceReferences` as an unsafe hammer on the **delete** path.
This ticket is the **import/overwrite** path reaching the same function, and it is worse there
because the delete path at least intends to destroy something: an import that overwrites a
texture has no reason to rewrite references at all — the `UTexture2D` object identity is
preserved by a re-import, so nothing needs replacing.

## Expected

1. **Do not call `ForceReplaceReferences` on a same-class re-import.** When the overwrite target
   and the newly imported object are the same `UObject` (or the same class at the same path),
   the reference set is unchanged; the commit stage should be a no-op. That alone removes this
   crash for the ordinary "re-import the texture I just regenerated" case, which is what every
   producer-driven workflow in this project does.
2. When a replace genuinely is needed (class changed, or the import produced a different object),
   use the same guarded path the delete tickets ask for, and **wrap the walk** so a poisoned
   referencer is reported rather than fatal.
3. Failing both, refuse with a typed error before importing: an `asset.import` that would run
   `ForceReplaceReferences` in an editor that has compiled a Blueprint this session is a coin
   flip on the whole process, and a refusal costs one caller a retry instead of costing every
   caller their session.

## Repro (shape, not yet minimised)

1. Shared editor, several agents working.
2. `blueprint.compile` and/or `blueprint.set_default` on any Blueprint.
3. `asset.import` a PNG over an **existing** texture asset path
   (`Docs/fps/data/textures/T_WPN_Markings_M.png` → `/Game/FPS/Weapons/Textures/T_WPN_Markings_M`).
4. Import reports success; ~6 s later the editor faults in `ForceReplaceReferences`.

Not minimised because reproducing it costs an editor. The log window above is complete and the
callstack is unambiguous; the ordering with the Blueprint compiles is the part that needs a
controlled run to confirm as *necessary* rather than merely present.

## Fix

The occupied-destination import path no longer stages, renames, deletes, garbage-marks, or
replaces the existing object, and it no longer calls `ObjectTools::ForceReplaceReferences`.
`overwrite:true` is admitted only for one exact, registry-identified `UTexture2D` and one bounded,
fully decoded PNG source. That lane strongly retains the existing object and a PinWright adapter
modeled on the UE 5.8 texture reimport factory, applies the requested source path, and passes that
exact adapter to automated `FReimportManager`; success is rejected unless the adapter itself ran.
The exact compressed bytes that passed preflight are the bytes imported. Postflight requires the
pointer, class, path, package, `StaticFindObject` result, and Asset Registry result to identify the
   same object, a changed texture-source content identity, a dirty package, and unchanged timestamp,
   size, and streaming hash for every pre-existing disk resource because `asset.import` never saves.
   Success reports
`mode:"inPlaceReimport"`, `identityPreserved:true`, `updatedInPlace:true`,
`contentChangeMeasured:true`, `textureContentChanged:true`, and `pendingSave`.

Every identity-changing, class-changing, multi-conflict, collateral, or unsupported occupied
case now returns `OVERWRITE_UNSAFE` before either production runner. Exact loaded but
registry-unregistered objects are still treated as occupied, while total destination entries,
matched packages, resource bytes, compressed PNG bytes, and decoded PNG bytes are bounded. The
error payload includes the requested and existing identity details, the refusal reason,
conservative conflicts, and a capped sorted Asset Registry referencer sample with its uncapped
   unique count. Every failure restores exact source metadata. A proven pre-adapter failure also
   restores prior package dirty state; after possible adapter mutation, failure or no-change leaves
   the package dirty and discloses that in-memory content may have changed. The safe identity-changing
   workflow remains caller-directed: import under a new
path, repoint known consumers, then delete the old asset through the unforced path.

## Reporter's position

**I was not the caller.** I was authoring `/Game/FPS/Weapons/Materials/M_WPN_OpticLens` in the
same editor and the crash ended my session; the import belonged to the markings stream. Filed
because an editor-killer that costs three other streams their work should not wait on the one
agent who happened to issue the call, and because the callstack lands squarely in
`AssetImportPolicy.cpp`. If the importing stream files its own, merge these.

## History
- `#1-forcereplacereferences-av-on-texture-reimport` `OPEN` reporter — First encounter, `2026-09-05T20:02:02Z`, UE 5.8. `asset.import` of `T_WPN_Markings_M.png` over the existing `/Game/FPS/Weapons/Textures/T_WPN_Markings_M`; Interchange reported the import complete at `20:01:56`, then `FScopedOverwriteStage::Commit` → `ObjectTools::ForceReplaceReferences` → `FFindReferencersArchive::ResetPotentialReferencer` → `UFunction::Serialize` → `FPropertyProxyArchive::operator<<` faulted with `EXCEPTION_ACCESS_VIOLATION reading 0xffffffffffffffff`. Crash report `UECC-Windows-01FD4B9D466127197DF55DB7729358DA_0002`; the same session id had already produced `_0000` and `_0001`. Distinct from `B-asset-import-unconditional-replace-hidden-outputs` (that ticket is about the overwrite being unconditional and multi-output results being hidden — it does not name a crash) and from the two `asset.delete` `ForceReplaceReferences` tickets (different verb, different stage). Ask: skip the replace entirely on a same-class re-import, where object identity is preserved and nothing needs replacing.
- `#2-bystander-cost-killed-a-critic-acceptance-slot` `OPEN` reporter — Bystander account of the
  SAME fault (20:02:02Z, `EXCEPTION_ACCESS_VIOLATION reading 0xffffffffffffffff` in
  `FPropertyProxyArchive::operator<<` under `UStruct::SerializeExpr` <- `ForceReplaceReferences`
  <- `AssetImportPolicy.cpp:1010` <- `AssetManageHandler.cpp:502`), filed to record the
  multi-agent cost rather than the mechanism, which `#1` already has right. I was the ENV critic
  mid-way through a 13-frame blind-A/B acceptance set under the world lock; the importing stream
  was a different one. Nine frames were captured, the tenth returned `EDITOR_NOT_RUNNING`
  (connection refused on 27145), and the slot, the lock and the remaining four frames plus
  z-fight, `level.audit` and the performance reading were lost. The interleaving is visible in
  the log: `render.capture_open_level` at 20:00:56 and 20:01:37 sit between the other stream's
  `blueprint.compile` (20:00:36, 20:01:45) and `blueprint.set_default` (20:01:06, 20:01:10)
  calls, so a read-only capture consumer was killed by an unrelated stream's write. Severity
  Critical is right: on a shared editor this verb's blast radius is every attached agent, not
  the caller. The lock file also had to be cleared by the coordinator because the holder died
  holding it, which is the second cost. No new mechanism, no separate file.
- `#3-caller-side-repro-source-citations-verified` `OPEN` reporter — **I am the caller `#1` asked for**, and I am confirming the ticket rather than filing the duplicate I was about to write. Same encounter, so `encounters`/`lastSeen` are left alone. **Exact call:** `asset.import { sourcePath: "X:/src/unreal/EAContentExamples58/Docs/fps/data/textures/T_WPN_Markings_M.png", destinationPath: "/Game/FPS/Weapons/Textures", overwrite: true }` — a 2048x512 8-bit grayscale PNG over an existing `UTexture2D` at `/Game/FPS/Weapons/Textures/T_WPN_Markings_M`. The RPC did not return a diagnosable error: it returned a transport failure (`stream read failed: [WinError 10054]`), because the process was gone. **The same call without `overwrite` had succeeded 8 minutes earlier** into an empty destination (`19:53:08:092` `Interchange start importing source [...T_WPN_Markings_M.png]`, `19:53:08:121` completed, saved `19:53:52`), so the crashing ingredient is the overwrite path, not the source file, the format, or the factory. Between the two imports, in the same editor and by me: `texture.set_texture_wrap`, `texture.set_compression_settings`, `property.set` on `SRGB`, `asset.save`, `material.compile_mgir` (Append, 140 expressions), `material.authoring.compile_material`, `material.authoring.set_material_instance_parameters`, `render.capture_asset_preview`, `editor.close_asset`. **Both `file:line` citations verified against the source in this checkout — do not take them on trust from the stack alone, one of the two is off by a line.** `Handlers/Asset/AssetManageHandler.cpp:502` lands exactly on `if (!OverwriteStage->Commit())`, inside the `asset.import` handler registered at `:294`. `Utils/AssetImportPolicy.cpp:1010` is **not** the `ForceReplaceReferences` call; it is `FAssetRegistryModule::AssetDeleted(ObjectEntry.Original.Get());`, i.e. the return address one statement past the call, which sits on **`:1009`**: `ObjectTools::ForceReplaceReferences(Current, Original);` where `Current = StaticFindObject(UObject::StaticClass(), nullptr, *ObjectEntry.OriginalPath)` (`:1006-1007`) and `TArray<UObject*> Original{ObjectEntry.Original.Get()}` (`:1008`). The enclosing loop is `Commit()`'s phase two, `:997-1018`. **Correction to `## Expected` item 1, and it is load-bearing: "skip the replace on a same-class re-import" cannot fire, because staging guarantees the two objects are never the same `UObject`.** `FScopedOverwriteStage::Stage()` (`:765`, moves at `:799-820`) calls `MoveAssetInMemory` to rename every pre-existing original into `/Game/__PinWrightAssetImportStage/<request-id>` **before** the factory runs, so the object the factory then creates at `OriginalPath` is a new one by construction. `Commit()`'s own gate (`:944-951`) counts `IsValid(Current) && Current != ObjectEntry.Original.Get()` and sets `PackageReplaced` when the count equals the object count — which for a plain re-import is always all of them. **So `ForceReplaceReferences` runs on every successful `overwrite:true` import, unconditionally; there is no path that skips it.** Fixing this needs either an in-place re-import (no move, so identity really is preserved) or a bounded referencer set, not an identity test that is structurally false. **Second structural finding: the code comment immediately above that loop is the bug in one line.** `:997-998` reads `// Phase two has no fallible operation: only after phase one succeeds do all / replaced originals lose rollback ownership together.` `ObjectTools::ForceReplaceReferences` is not merely fallible — it took the process down. The whole staging/rollback contract (`staged[]`, `restored[]`, `removedCreated[]`, `restoreSucceeded`, the 64-package / 512 MiB fail-closed bounds) is placed in phase one on the belief that phase two cannot fail, and a process kill in phase two makes every one of those guarantees unreachable: none of them ran, and none of them could have. **One hypothesis ruled out:** `FImpl::FObjectEntry::Original` is a `TStrongObjectPtr<UObject>` (`:236`), so the staged original cannot be collected or nulled — the dangling `FProperty*` is not the staged object but a referencer the whole-graph iterator reached, exactly as `## Mechanism` says. **Timing, which sharpens the concurrency reading in `#1`:** the fault is in frame `779` and Interchange finished in the same frame at `20:01:56:368`, so `ForceReplaceReferences` ran for **~6.6 s inside a single frame**, inside the one core-ticker safe-point hop `LogPinWrightSafePoint` announced at `20:01:56:223` — a whole-object-graph walk per staged object, not a bounded fix-up. The three Blueprint compiles were `blueprint.set_default` on `/Game/FPS/Weapons/BP_WeaponBase` (`20:01:06:265`), `blueprint.set_default` on `BP_Weapon_Pistol` (`20:01:10:261`) and `blueprint.compile` on `BP_Weapon_AR` (`20:01:45:558`, `LogUObjectHash: Compacting FUObjectHashTables data took 0.76ms` at `:592`). All three ran **inline** on their own frames (630, 642, 747) and all had **completed** before the import's frame 779 began. **State this as a hypothesis, not a cause, because that is what the evidence supports:** the compiles were not concurrent with the walk, so the candidate mechanism is not a race but *residue* — reinstancing leaves `REINST_`/`TRASHCLASS_` `UFunction` objects alive in the graph until a GC that had not happened, and `BP_WeaponBase` is a parent class, so its recompile reinstanced its children too. The whole-graph `FThreadSafeObjectIterator` walk reaches those trashed functions and serialises their bytecode. **The test that would settle it** and that nobody has run: `asset.import` with `overwrite:true` immediately after a `blueprint.compile` in an otherwise idle editor, versus the same import in an editor where no Blueprint has been compiled since the last GC — if only the first faults, the residue reading is right and a forced `CollectGarbage` before the commit is a candidate mitigation; if both fault, the walk itself is the defect and the compiles are noise. Minimising it costs an editor, which is why I have not done it. **Third, documentation-level problem, worth its own line in a fix:** `Saved/PinWright/wiki/asset.import.md` is 26 lines and describes the overwrite path at length — staging, the request-scoped stage namespace, atomic file replacement, rollback, `restoreSucceeded`, `rolledBack`, the 64-package and 512 MiB fail-closed bounds — and **the word "reference" does not appear anywhere on the page**, nor does `ForceReplaceReferences`, nor any hint that committing an overwrite walks every live `UObject` in the editor. The documented risk of `overwrite:true` is "the wrong packages get replaced", a bounded, recoverable, caller-scoped risk the page tells you how to reason about. The actual risk is "the editor dies and every other agent in it loses its session". A caller who reads that page carefully is *more* likely to use `overwrite:true`, not less. **On-disk state after the crash, checked because the rollback machinery never got to run:** `Content/FPS/Weapons/Textures/T_WPN_Markings_M.uasset` is **intact and not half-written** — 25608 bytes, mtime `2026-09-05 22:56:08.417 +0300` = `19:56:08 UTC`, matching the log's last `LogSavePackage: Moving output files for package: /Game/FPS/Weapons/Textures/T_WPN_Markings_M` at `19:56:08:417`, six minutes *before* the crashing import. Header magic `c1 83 2a 9e` and the trailing package tag `c1 83 2a 9e` are both present, and there are no `.uexp`/`.ubulk` siblings missing. So the file on disk is the earlier good save, byte-for-byte unchanged by the failed import; what was lost is only the new texture, which existed solely in memory (`LogTexture: Building texture TwoD ... TFO_AutoDXT` at `20:01:56:247`, never saved). No stray `Content/__PinWrightAssetImportStage` directory exists on disk either, consistent with the stage packages being in-memory only and never having been asked to serialise. That is luck, not design: this import happened to be a single-package overwrite whose commit died before any file write. A multi-package overwrite that faults after phase one has retired some stage packages and before others would leave exactly the mixed state `restoreSucceeded` exists to report and cannot, because there is no process left to report it. Crash artefact `Saved/Crashes/UECC-Windows-01FD4B9D466127197DF55DB7729358DA_0002` (`ErrorMessage: Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0xffffffffffffffff`), full log preserved at `Saved/Logs/EAContentExamples58-backup-2026.09.05-20.02.03.log` lines 9702-9742 — the live `EAContentExamples58.log` has since rotated, so quote the backup. **Workaround I am using, with its caveat:** do not pass `overwrite: true` at all. Either import under a new name and repoint consumers, or `asset.delete` the existing asset first and import into the now-empty destination — but note that `asset.delete`'s `force:true` path reaches this same `ObjectTools::ForceReplaceReferences` (`B-force-delete-nulls-referencers`), so only the *unreferenced* delete, or the new-name route, actually avoids the hazard. There is no in-band way for a caller to know an overwrite is about to kill the process, and no return value to check: the RPC dies as a socket error.
- `#4-blast-radius-is-two-verbs-unforced-delete-measured-safe` `OPEN` reporter — **The blast radius is two verbs, not one, and the unforced third route is measured safe.** Same encounter as `#1`-`#3` — added analysis, not a new sighting, so `encounters` / `lastSeen` are left alone. `asset.import {overwrite: true}` reaches `ObjectTools::ForceReplaceReferences` through `AssetImportPolicy::FScopedOverwriteStage::Commit()` → `FFindReferencersArchive::ResetPotentialReferencer()` → `UFunction::Serialize()` → `UStruct::SerializeExpr()` → `FPropertyProxyArchive::operator<<()` → `EXCEPTION_ACCESS_VIOLATION reading 0xffffffffffffffff` (this ticket's `## Symptom` stack). **`asset.delete {force: true}` reaches the same function, over the same object graph, in the same editor.** Quoted verbatim from `Saved/PinWright/wiki/asset.delete.md` and checked against the page rather than taken on trust: "`UEditorAssetLibrary::DeleteAsset` is `ObjectTools::ForceDeleteObjects`, which runs `ForceReplaceReferences(nullptr, ...)` over **every live UObject in the editor** *before* the engine decides whether the package may go". The only difference between the two verbs at that call is the argument — the import path passes a replacement object (`ObjectTools::ForceReplaceReferences(Current, Original)`, `AssetImportPolicy.cpp:1009`, cited in `#3`), the delete path passes `nullptr` — and the fault in `## Mechanism` is in the *walk*, not in what is substituted, so `force: true` is exposed to the identical dangling-`UFunction` hazard. Sibling ticket for that verb: `B-force-delete-nulls-referencers`. **Positive control, measured this session: the UNFORCED `asset.delete` path is safe.** Two calls, both `{path: "<texture>"}` with no `force`, on textures nothing referenced any more. `/Game/FPS/Weapons/Textures/T_WPN_Markings_M` → `success: true`, `deletedCount: 1`, `existsAfter: false`, `existsOnDisk: false`, `deleteReported: true`, `referencingBlueprints: []`; `/Game/FPS/Weapons/Textures/T_WPN_FlankAO_M` → same shape, same result. The editor survived both and kept serving RPCs (subsequent `asset.save` calls returned `saveState: "written"`), and both `.uasset` files are gone from `Content/FPS/Weapons/Textures/` on disk. The wiki gives the reason: the unforced path routes through `ObjectTools::DeleteObjects` → `FAssetDeleteModel`, never through `ForceDeleteObjects`, so it does not reach `ForceReplaceReferences` at all. **Caller guidance, which is the part every stream needs:** to replace an asset's contents, do **not** reach for `overwrite: true`, and do **not** reach for `force: true` when the unforced delete refuses with `ASSET_IN_USE`. Import under a **new** asset path, repoint the referencing material / mesh / Blueprint at it, and only then `asset.delete` the old one unforced — by that point it is unreferenced and the safe path succeeds. That is the sequence that worked here: `T_WPN_MarkingsPlate_M` and `T_WPN_FlankOcclusion_M` imported under fresh names, `material.compile_mgir` repointed `M_WPN_Master` at them, then both superseded textures deleted unforced with no incident. **`ASSET_IN_USE` is therefore a useful refusal**, not an obstacle to route around: it costs one caller a retry, whereas reaching for `force` to clear it converts a refused call into a possible process kill that takes every other agent in a shared editor with it — the cost `#2` records from the other side.
- `#5-same-identity-texture-reimport-no-global-walk` `IN-REVIEW` developer — Removed the staged identity-replacement commit and both import-path global reference walks. Occupied imports now admit only the proven one-object `UTexture2D` same-identity reimport lane; all other occupied overwrite shapes fail before import with `OVERWRITE_UNSAFE`. Added the focused `SavedLoadedOverwritePreservesLiveReferences`, `OverwriteFalseRefusesSavedLoadedAsset`, `PostStageFailureRestoresSavedLoadedAsset`, and `ProductionMultiOutputReportsEveryOutput` contracts. No build, editor, automation, or MCP run was performed in this implementation stream; the focused run of these tests will be their first execution.
