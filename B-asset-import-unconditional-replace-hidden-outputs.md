---
id: B-asset-import-unconditional-replace-hidden-outputs
title: "asset.import unconditionally replaces existing assets and reports only the first object produced by a potentially multi-output import"
status: IN-REVIEW
severity: Critical
category: bug
tags: [asset, import, overwrite, data-loss, partial-result, false-success]
---

# `asset.import` can overwrite content without opt-in and hides collateral outputs

## What's wrong

`AssetManageHandler.cpp:290-297` always sets `UAutomatedAssetImportData::bReplaceExisting = true`
before calling `ImportAssetsAutomated`. The schema has no `overwrite`/`replace` parameter and the
handler never checks whether the destination package already exists. A repeated import can
therefore replace a hand-edited asset with no refusal, backup, or warning.

The same call can return several objects, but `:299-308` selects the first non-null object and
discards every other identity. It then optionally renames that one object and publishes one
`assetPath` (`:310-342`). A factory that creates multiple assets can overwrite or create more
packages than the response admits. If the rename fails, the response is still success and the
published path is the requested path even though destination verification is optional.

## What it should do

Default to `overwrite:false` and refuse every pre-existing target before replacement. Return a
measured `results[]` for every imported object, including its canonical path, whether it replaced
something, rename outcome, and persistence state. An explicit overwrite must preserve or stage
the old package until all outputs are verified.

## Workaround

Import into a new empty folder and use `asset.list` to discover all produced assets before moving
them into final locations.

## Fix

Verdict: valid bug, wrong staged fix. `asset.import` keeps `overwrite:false` as the default and
refuses an occupied conservative source/request-prefix destination before either runner. Empty
destinations still use the real `IAssetTools::ImportAssetsAutomated` path. Its ordered `results[]`
contains every non-null primary and `UFactory::AdditionalImportedObjects` output with its actual
canonical path, primary-only rename metadata, created/update measurement, and persistence state.

An occupied destination is admitted only when it is one exact, registry-identified `UTexture2D`
and the source is one bounded, fully decoded PNG. Production strongly retains the existing pointer
and a PinWright adapter modeled on UE 5.8's texture reimport factory, applies the requested source
path, and passes that exact adapter to `FReimportManager`; success is rejected unless the adapter
itself ran. The adapter imports the exact compressed bytes that passed preflight. Success requires
the pointer, class, path, package, `StaticFindObject`, and Asset Registry resolution to remain
identical, texture-source content identity to change, the package to be dirty, and every captured
   disk resource to retain its timestamp, size, and streaming hash because the verb does not save.
   The response reports
`mode:"inPlaceReimport"`, `identityPreserved:true`, `updatedInPlace:true`,
`contentChangeMeasured:true`, `textureContentChanged:true`, and `pendingSave`, with
`replaced:false`.

Every identity-changing, class-changing, multi-conflict, collateral, rename-over-occupied, or
unsupported occupied case returns `OVERWRITE_UNSAFE` before import. A loaded but
registry-unregistered exact object is still occupied and cannot fall through to AssetTools.
Destination entries, matched packages, resource bytes, compressed source bytes, and decoded bytes
are bounded; disk resources use streaming hashes, and referencer diagnostics return a capped sorted
sample plus the uncapped unique count. There are no stage moves, import-path global reference
   walks, automatic reference fixups, or generic atomic content rollback claims. Every failure
   restores exact source metadata. A proven pre-adapter failure also restores prior dirty state;
   after possible adapter mutation, failure or no-change leaves the package dirty and reports that
   content may have mutated. The safe replacement workflow is explicit and
caller-directed: import under a new path, repoint known consumers, then delete the old asset
unforced.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the data-loss and false-success catalogs; no editor, build, test, or RPC run was performed.
- `#2-safe-default-complete-results` `IN-REVIEW` developer — Made overwrite opt-in with a false default, refused an occupied destination without overwrite, emitted measured results for every imported object, made rename failure return an error with actual paths, documented non-atomic explicit overwrite, and added `PinWright.asset.import.SafeDefaultAndCompleteResults`. Source inspection only; no build, editor, automation, or MCP run.
- `#3-staged-overwrite-restoration` `IN-REVIEW` developer — Restored the original staged-overwrite requirement after review, staged every destination-prefix package before explicit overwrite, restored staged packages after injected import and verification failures, replaced inferred replacement with identity/hash evidence, documented the prefix bound, and added handler-level occupied-destination and rollback coverage. Source inspection only; no build, editor, automation, or MCP run.
- `#4-returned` `OPEN` reporter — **The `#3` staged-overwrite path crashed the shared editor.** `AssetImportPolicy::FScopedOverwriteStage::Commit()` took an `EXCEPTION_ACCESS_VIOLATION reading address 0xffffffffffffffff` on EAContentExamples58 (UE 5.8) at `2026.09.05-20.02.02Z`, killing the process and every agent's unsaved work in it. `#3` recorded "Source inspection only; no build, editor, automation, or MCP run" — this is its first run in an editor.

  Callstack, top-down from the plugin frame (`Saved/Logs/EAContentExamples58.log`):

  ```
  UnrealEditor-PinWright.dll!AssetImportPolicy::FScopedOverwriteStage::Commit()  AssetImportPolicy.cpp:1010
  UnrealEditor-UnrealEd.dll!ObjectTools::ForceReplaceReferences()                ObjectTools.cpp:1504
  UnrealEditor-UnrealEd.dll!ObjectTools::ForceReplaceReferences()                ObjectTools.cpp:1369
  UnrealEditor-UnrealEd.dll!`ObjectTools::ForceReplaceReferences'::<lambda_1>()  ObjectTools.cpp:1314
  UnrealEditor-CoreUObject.dll!FFindReferencersArchive::ResetPotentialReferencer() FindReferencersArchive.cpp:89
  UnrealEditor-CoreUObject.dll!UFunction::Serialize()                            Class.cpp:7608
  UnrealEditor-CoreUObject.dll!UStruct::Serialize()                              Class.cpp:2458
  UnrealEditor-CoreUObject.dll!UStruct::SerializeExpr()                          Class.cpp:2691
  UnrealEditor-CoreUObject.dll!FPropertyProxyArchive::operator<<()               PropertyProxyArchive.h:46
  ```

  The request was `asset.import` of `Docs/fps/data/textures/T_WPN_Markings_M.png` over the existing `/Game/FPS/Weapons/Textures/T_WPN_Markings_M` (log `20.01.56` -> `20.02.02`, 6 s from import start to crash), issued by a different stream in the shared editor.

  **What the stack says.** `FFindReferencersArchive` walks *every* `UObject` in memory to find referencers of the staged originals, and `UFunction::Serialize` -> `SerializeExpr` means it was walking a **UFunction's compiled script bytecode** and dereferencing a property pointer that was `0xffffffffffffffff`. So the commit's referencer sweep is not scoped to the import's own package graph — it reads the bytecode of unrelated classes, and it faulted on one.

  **Co-occurrence, offered as a hypothesis and not as a measurement I can prove.** My stream (WEAPONS blueprints) had been recompiling `/Game/FPS/Weapons/BP_WeaponBase` and `/Game/FPS/Weapons/BP_Weapon_AR` in that same editor throughout the preceding half hour, the last one 17 s before the fault (`blueprint.compile` of `BP_Weapon_AR`, log `20.01.45`), and those classes were recompiled-but-unsaved at the moment of the crash. A whole-heap bytecode walk concurrent with Blueprint reinstancing is a plausible source of a stale property pointer in `SerializeExpr`. I did not reproduce it and I am not asserting the causal link — but the fix should not need me to, because a texture reimport has no business serializing an unrelated Blueprint's bytecode at all.

  **What should change.** Scope the commit's referencer replacement to the packages the import actually staged, rather than calling the global `ObjectTools::ForceReplaceReferences` over the whole object graph — the same objection `B-force-delete-nulls-referencers` raises against the sibling `ObjectTools::ForceDeleteObjects` path, and for the same reason: these engine helpers are editor-UI-scoped operations with no isolation from other work in flight. Failing that, `#3` needs the editor run it never had, with a Blueprint compile in flight, before it goes back to IN-REVIEW.

  **Cost to this stream:** every `blueprint.graph.*` mutation is `pendingSave:true` until an explicit `asset.save`, so ~35 minutes of committed-to-memory graph surgery on `BP_WeaponBase` (penetration budget, falloff rewire, instigator, tracer phase, dry-fire gate, `CancelReload` contract) died with the process. Only the three `blueprint.add_variable`/`add_dispatcher` results, which save themselves, reached disk.

  **Cross-reference, added after filing:** the importing stream filed the crash itself as `B-asset-import-overwrite-commit-kills-editor` (Critical, `encounters: 2`) with the same log window and callstack. That ticket owns the crash; **this entry exists for the separate claim that `#3-staged-overwrite-restoration` must not stay `IN-REVIEW`**, since the code it added is the code that faulted and it was never run in an editor. Fix one, check the other before closing either.
- `#5-safe-default-complete-results-same-identity-reimport` `IN-REVIEW` developer — Replaced the returned staged-overwrite design with a narrow same-object `UTexture2D` reimport lane and pre-import `OVERWRITE_UNSAFE` for every other occupied shape. Preserved real empty-destination AssetTools import and added a deterministic real-factory regression proving ordered primary plus `AdditionalImportedObjects` reporting, alongside the saved-loaded live-reference overwrite/refusal/failure tests. No build, editor, automation, or MCP run was performed in this implementation stream; the focused run of these four tests will be their first execution.
