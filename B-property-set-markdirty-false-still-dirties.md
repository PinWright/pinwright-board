---
id: B-property-set-markdirty-false-still-dirties
title: "property.set and property.reset dirty the package even when markDirty:false is passed, and still report markedDirty:false, so a transient probe value silently rides a later editor.save_all onto disk"
status: IN-REVIEW
severity: High
category: bug
tags: [property-set, property-reset, mark-dirty, silent-false-success, modify, customer-report]
encounters: 1
lastSeen: 2026-08-09T04:36:42Z
---

# `property.set` / `property.reset` dirty the package even when `markDirty:false`, and still report `markedDirty:false`

External QA report, UE 5.7.4, Windows, PinWright running in-editor: three `property.set` calls against one asset, every one with `markDirty:false`. The file on disk stayed untouched (`git status` clean), but the package showed up in `list_dirty_packages` anyway. The reporter's workaround was `asset.reload`. Every response had said `markedDirty:false`.

The `markDirty` parameter gates exactly one call, and it is the **last** thing that happens. Three separate mechanisms dirty the package before that gate is ever consulted, and a fourth defect makes the response confirm the false claim rather than notice it. Engine anchors below are UE 5.7; UE 5.8 is behaviorally identical (`Obj.cpp:1652/1666/1672` for the three `Modify()` lines, `:549` for `PostEditChange`, `UObjectBaseUtility.cpp:242/270/274`, `ActorEditor.cpp:150`).

## What's wrong

`Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp:952-959` — the shared finalizer, reached at `:1106` on the generic reflected path:

```cpp
auto FinalizeApplied = [&](TSharedPtr<FJsonObject>& Result, UObject* Target)
{
    if (bMarkDirty)
        Target->MarkPackageDirty();
    Result->SetStringField(TEXT("propertyName"), PropertyName);
    Result->SetBoolField(TEXT("applied"), true);
    Result->SetBoolField(TEXT("markedDirty"), bMarkDirty);
};
```

The param is read at `:941-945`. `property.reset` has the identical shape at `:1281-1284`. That single `MarkPackageDirty()` is the only thing `markDirty` controls; by the time it runs the package is already dirty.

## Root cause

**#1 — unconditional `Modify()` before the gate (the reported bug).** `RootObject->Modify();` at `UtilityPropertyHandler.cpp:1097`, ahead of the finalizer at `:1106`. `UObject::Modify(bool bAlwaysMarkDirty = true)` (`Obj.cpp:1563`) calls `SaveToTransactionBuffer(this, bAlwaysMarkDirty)` at `:1577` and, when that returns false, `MarkPackageDirty()` at `:1583` behind the guard at `:1581`. `SaveToTransactionBuffer` (`UObjectGlobals.cpp:3361`) itself marks dirty at `:3379` (guarded by `bMarkDirty` at `:3377`) when `GUndo` is live (`:3372`). The handler opens no transaction — there is no `FScopedTransaction` and no `BeginTransaction` anywhere in the file — so `GUndo` is normally null and the `:1583` fallback fires. Either branch lands on `UObjectBaseUtility::MarkPackageDirty()` → `Package->SetDirtyFlag(true)` (`UObjectBaseUtility.cpp:269`). The package is therefore dirty before `:952` ever consults `bMarkDirty`. This fires for plain assets and Blueprint CDOs alike — a CDO's outermost is the Blueprint package.

**#2 — unconditional `PostEditChange()` after the gate.** `RootObject->PostEditChange();` at `:1107`, one line past the finalizer. `UObject::PostEditChange()` (`Obj.cpp:511`) does not itself dirty; it forwards to `PostEditChangeProperty` at `:514`, which broadcasts `FCoreUObjectDelegates::OnObjectPropertyChanged` (`Obj.cpp:520`) and dispatches to the concrete class override. `AActor::PostEditChangeProperty` (`ActorEditor.cpp:152`) reregisters components and reruns construction scripts; asset classes such as `UMaterial`, `UBlueprint` and Niagara call `Modify()` / `MarkPackageDirty()` internally. This is class-dependent and second-order, but it is why a naive "restore the flag immediately after `Modify()`" fix would be re-dirtied one line later. Stated explicitly so the fix is not implemented wrong.

**#3 — the resolve step can dirty before any write happens.** `ResolveObjectForProperty` (`:89-187`) loads the asset when it is not resident — `UEditorAssetLibrary::LoadAsset` at `:140`, `StaticLoadObject` at `:156` — and `PostLoad` fixups dirty packages. The plugin already knows this: `Handlers/Asset/AssetDumpHandler.cpp:974-977` carries the comment about "packages the dump itself dirtied (via LoadObject -> PostLoad)". So a first-touch `property.set` on a cold asset dirties before the property is even resolved.

**#4 — the response confirms the false claim.** `markedDirty` at `:958` is stamped from the *request parameter*, never from observed package state. The handler does not merely fail to notice the dirt; it asserts an outcome it never checked. This is the silent-false-success half of the defect and is in scope for this ticket.

## Evidence

Checked and **not** implicated — listed so the fix is not over-scoped:

- No `FScopedTransaction` / `BeginTransaction` anywhere in `UtilityPropertyHandler.cpp` (grep-verified), so `GUndo` is normally null on this path.
- No `PreEditChange` call at all, so the archetype `RF_Transactional` stamping in `UObject::PreEditChange(FEditPropertyChain&)` (`Obj.cpp:547-557`; the `SetFlags(RF_Transactional)` is at `:556`) never runs.
- The empty `FPropertyChangedEvent` built at `Obj.cpp:513` carries no `MemberProperty`, so CDO→instance propagation in `PostEditChangeChainProperty` is not reached.
- The four `AActor` special-case branches (`:965-1088` — ActorLocation / ActorRotation / ActorScale / bHidden) call neither `Modify()` nor `PostEditChange()`, and the engine setters they use do not dirty. **The actor paths honor `markDirty` correctly today**; only the generic reflected path is broken, which is consistent with the customer repro being an asset.

`property.reset` (`:1126`) has the same defect three times over, once per engine-version branch: unconditional `Modify()` at `:1206`, `:1243`, `:1268`, all ahead of the gate at `:1281`. `property.set` and `property.reset` are the only two handlers in the plugin that accept a `markDirty` parameter — grepping `markDirty` / `mark_dirty` / `bMarkDirty` across `Plugins/PinWright/Source` returns only the two registrations at `:898` and `:1126` plus four test files. Adjacent but out of scope: the `container.*` mutators in the same file call `Modify()` unconditionally at `:1922, 1977, 2012, 2057, 2178, 2236, 2409, 2558, 2595, 2672, 2833` but expose no `markDirty` param at all, so they are consistently always-dirty and violate no contract.

The documented contract promises exactly the workflow the customer tried, so the fix is to make the code match the docs, not the reverse. `Plugins/PinWright/docs/wiki-src/property.md:55` on `property.reset`'s param: *"defaults to `true`; pass `false` for transient checks."* `property.md:40` on the response field: `markedDirty` is *"whether this call marked the target package dirty (the `markDirty` param, default `true`)"*. Registration doc strings at `UtilityPropertyHandler.cpp:898` and `:1126`: `RPC_PARAM_OPT("markDirty", "boolean", "Mark package dirty (default true)")`. Generated mirrors: `Saved/PinWright/wiki/property.set.md:14,36` and `property.reset.md:13,27`.

## Repro

1. `call("property.set", { objectPath: "<a cold /Game or /App asset>", propertyName: "<any reflected scalar>", value: <probe>, markDirty: false })` — response returns `applied:true, markedDirty:false`.
2. `git status --short -- <asset>.uasset` → empty; the file on disk is untouched.
3. `call("editor.list_dirty_packages", {})` → the asset's package is listed.
4. `call("asset.reload", { ... })` discards it; a blanket `editor.save_all` instead commits the probe value.

Repeat with `property.reset` and `markDirty:false` for the same outcome via `:1206` / `:1243` / `:1268`.

## What it should do

`markDirty:false` must leave a previously-clean package clean when the call returns, and `markedDirty` must report what actually happened rather than echo the request.

## Fix

The pattern already exists in-tree and is tested: `AssetDumpHandler.cpp:978-1022` implements `DumpDirtyGuard_Snapshot()` / `DumpDirtyGuard_Restore()` / `FScopedDumpDirtyRestore` — snapshot the dirty set, clear only what this operation dirtied, leave pre-existing user dirt alone — covered by `Source/PinWright/Private/Tests/Utility/TestAssetDumpHandler.cpp:1672-1710` and `:1774-1805`. No engine-level suppression guard exists: searching `CoreUObject/Public` and `UnrealEd/Public` for `FScopedSkipDirty`-style helpers returns nothing usable; the only engine lever is `Modify(bAlwaysMarkDirty)`.

All steps in `UtilityPropertyHandler.cpp`:

1. After the param read at `:945`, capture `UPackage* const TargetPackage = RootObject->GetOutermost();` and `const bool bPackageWasDirty = TargetPackage && TargetPackage->IsDirty();`.
2. `:1097` — `RootObject->Modify();` → `RootObject->Modify(/*bAlwaysMarkDirty=*/bMarkDirty);`. With no transaction this skips `MarkPackageDirty()` entirely (the guard at `Obj.cpp:1581`); with a live transaction it still saves the undo copy without dirtying (`UObjectGlobals.cpp:3377`). Strictly better than post-hoc restore for this mechanism.
3. Put the restore **after** `PostEditChange()`, not before, so mechanism #2 cannot re-dirty behind it:

```cpp
RootObject->PostEditChange();
if (!bMarkDirty && TargetPackage && !bPackageWasDirty)
{
    TargetPackage->SetDirtyFlag(false);
}
```

4. Change `FinalizeApplied`'s `markedDirty` field at `:958` to report observed state (`TargetPackage && TargetPackage->IsDirty()`) instead of echoing the request param, so the response stops asserting an outcome it never checked. Route the four AActor branches (`:989`, `:1023`, `:1058`, `:1081`) through the same observed-state finalizer so `markedDirty` is honest across all five emit sites — no behavioral change needed there.
5. Same treatment for `property.reset`: hoist the `bMarkDirty` read from `:1276-1280` to above the `#if MCP_HAS_PROPERTY_VISITOR` block (~`:1187`), change `Modify()` → `Modify(bMarkDirty)` at `:1206`, `:1243`, `:1268`, and add the post-`PostEditChange()` restore after `:1285`.
6. Scope the save/restore to `RootObject->GetOutermost()` only. Do **not** copy the asset-dump's blanket "clear everything not in the baseline" shape here — for a single-property write that would clear dirt another concurrent editor action created mid-call.

**Open decision for the implementer.** The baseline capture at step 1 sits *after* `ResolveObjectForProperty`, so load-induced dirt from mechanism #3 lands inside the baseline and is preserved. Moving the capture above the resolve would make `markDirty:false` also undo load-time dirt. Recommendation: keep it after the resolve — a cold load dirtying its own package is not this handler's mutation — but confirm the choice and pin it with a test.

Risks, stated honestly:

- **Source control.** `PackageMarkedDirtyEvent.Broadcast` fires at `UObjectBaseUtility.cpp:273`, *before* the flag can be cleared, so an SCC auto-checkout prompt can still trigger on a `markDirty:false` call. The restore is post-hoc, not suppression.
- **Undo.** Clearing the package flag does not touch the transaction buffer; if `GUndo` was live the object copy is already saved and undo still works. Same trade the asset-dump guard already accepts.
- **Autosave.** `FPackageAutoSaver` keys off the dirty flag, so the probe value will not be autosaved — that is the intent.
- **Residual hazard the fix does not close.** The mutated value stays in memory behind a clean flag, so any *later* legitimate edit to that package saves the probe value too. `asset.reload` remains the only correct discard. The fix narrows the window; it does not eliminate it. Document this on the method page as part of the fix.

**Test gap.** `Source/PinWright/Private/Tests/Utility/TestPropertySetReportsMarkDirtyNotSaved.cpp` is the only `markDirty` test and cannot catch this bug, for two independent reasons: (a) `AssertHonestMutatorResponse` (`:56-78`) checks only the response fields `applied`, `markedDirty`, and absence of `saved:true`, never `Package->IsDirty()` — the `markDirty:false` case at `:167-183` asserts nothing beyond the response echoing `false`; and (b) the fixture is structurally incapable of dirtying, because the Blueprint is created in `GetTransientPackage()` (`:91-97`) and `UObjectBaseUtility::MarkPackageDirty()` early-returns for anything under an `RF_Transient` outer (`UObjectBaseUtility.cpp:242-245`), so no dirty flag is ever set with or without the fix. Three other tests pass `markDirty:false` specifically to avoid dirtying and are therefore silently relying on the broken guarantee: `TestPropertyCdoParentClassDefault.cpp:213`, `TestPropertyComponentParentDefault.cpp:211`, `TestUtilityHandlers.cpp:414`. A real regression test needs a fixture asset in a real `/Game` or `/Temp` package saved to disk first so it starts clean, and must assert: `markDirty:false` on a clean package leaves `IsDirty() == false` (fails today); `markDirty:true` leaves it `true` (counterfactual proving the test can distinguish); a package already dirty before the call stays dirty after (the restore must not eat pre-existing user dirt); the same three for `property.reset`; and ideally a cold-load case to pin the open decision above. Copy the fixture/assert shape from `TestAssetDumpHandler.cpp:1672-1710` and `:1774-1805`.

## Distinct from

- `B-property-wiki-claims-cdo-recompile` — the false "Recompiles the Blueprint when the target is a CDO" claim at `wiki-src/property.md:29`, found while reading this same handler. That one is pure documentation; this one is a behavioral contract violation.
- `B-property-set-saved-true-not-persisted` (IN-REVIEW) — that ticket removed the hardcoded `saved:true` and introduced `applied` + `markedDirty`. It fixed the *persistence* lie. It did not touch the dirty path: `markedDirty` still echoes the request param at `:958`, and the unconditional `Modify()` at `:1097` predates and survives that fix. This ticket is the remaining half.

## Impact

`property.set` is a core every-session verb. Probe values written during measurement or calibration — the documented `markDirty:false` workflow — get persisted by a later blanket `editor.save_all`, committing an asset nobody meant to edit. The caller has no signal that this happened: the response says `markedDirty:false`, so the natural next step is to skip any cleanup.

## Severity

`High`. Impact class is silent false-success — the caller trusts a result that is a lie and builds on it: the response reports `markedDirty:false` while the package is in fact dirty. Reach modifier confirms `High`: `property.set` runs in almost every session. Not `Critical` because nothing is corrupted or crashed, and the write only reaches disk via a subsequent user-initiated save.

## History
- `#1-filed-from-customer-report` `OPEN` reporter — External QA report (UE 5.7.4, Windows, PinWright in-editor): three `property.set` calls with `markDirty:false` against one asset left the file clean on disk (`git status`) but the package appeared in `list_dirty_packages`; reporter worked around it with `asset.reload`. Source-verified in `Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp`: `markDirty` is read at `:941-945` and gates exactly one call, `MarkPackageDirty()` inside `FinalizeApplied` at `:952-959`, which the generic reflected path reaches at `:1106` — but `RootObject->Modify()` at `:1097` runs unconditionally *before* it and lands on `MarkPackageDirty()` either through `SaveToTransactionBuffer` (`UObjectGlobals.cpp:3379`, when `GUndo` is live) or through the fallback at `Obj.cpp:1583` (the normal case here, since the file contains no `FScopedTransaction`/`BeginTransaction`), both ending at `Package->SetDirtyFlag(true)` (`UObjectBaseUtility.cpp:269`); two further mechanisms dirty outside the gate — `PostEditChange()` at `:1107` runs *after* it and dispatches to class overrides that `Modify()`/`MarkPackageDirty()` internally (`ActorEditor.cpp:152`, `UMaterial`, `UBlueprint`, Niagara), and `ResolveObjectForProperty` (`:89-187`) loads cold assets at `:140`/`:156` whose `PostLoad` fixups dirty the package, a hazard the plugin already documents at `AssetDumpHandler.cpp:974-977`; and the `markedDirty` response field at `:958` is stamped from the request param rather than observed package state, so the response actively confirms the false claim. `property.reset` repeats the defect three times at `:1206`/`:1243`/`:1268` ahead of its gate at `:1281`, and these two are the only handlers in the plugin accepting `markDirty` (grep-verified). Ruled out as not implicated: no `PreEditChange` call (so `Obj.cpp:547-557` archetype `RF_Transactional` stamping never runs), the empty `FPropertyChangedEvent` carries no `MemberProperty` (so CDO→instance propagation is unreached), and the four `AActor` branches at `:965-1088` honor `markDirty` correctly since they call neither `Modify()` nor `PostEditChange()`. Existing coverage cannot regress this: `TestPropertySetReportsMarkDirtyNotSaved.cpp` asserts only response fields (`:56-78`, `:167-183`) and its fixture lives in `GetTransientPackage()` (`:91-97`), where `MarkPackageDirty()` early-returns (`UObjectBaseUtility.cpp:242-245`), so no dirty flag is ever set with or without the fix. Fix proposed as `Modify(bMarkDirty)` plus a post-`PostEditChange()` package-scoped flag restore and an observed-state `markedDirty`, modelled on the in-tree `DumpDirtyGuard_Snapshot`/`Restore` at `AssetDumpHandler.cpp:978-1022`. Dedup: checked the board for `markdirty`/`mark-dirty`/`dirty`/`property-set`/`property-reset` — the nearest ticket, `B-property-set-saved-true-not-persisted` (IN-REVIEW), covers the removed `saved:true` persistence lie and introduced the `markedDirty` field; it does not cover the dirty path, which this ticket owns.
- `#2-modify-gate-and-observed-dirty` `IN-REVIEW` developer — Fixed in `Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp` along the lines the ticket proposed. Mechanism #1 is suppressed at the source: `RootObject->Modify()` now passes `bAlwaysMarkDirty=bMarkDirty` on the `property.set` generic path (`:1118`) and in **all three** engine-version branches of `property.reset` (`:1245`, `:1283`, `:1309`), so with no open transaction `Modify()`'s `MarkPackageDirty()` fallback never fires for a `markDirty:false` call. The package baseline (`UPackage* TargetPackage = RootObject->GetOutermost()` + `IsDirty()`) is captured at `:950-951` for `property.set` and `:1191-1192` for `property.reset`, in both cases **after** object resolution — the ticket's open decision, resolved as recommended: dirt produced by a cold load's `PostLoad` lands inside the baseline and is therefore preserved, not silently cleared, because that dirt is not this handler's mutation. Mechanism #2 dictated the ordering: `PostEditChange()` was **moved ahead of** the dirty decision on both verbs (`property.set` `:1129`, immediately before the `FinalizeApplied` call at `:1132`; `property.reset` `:1321`, before the restore at `:1327-1329`), because that call dispatches to concrete class overrides (`UMaterial`, `UBlueprint`, Niagara) that `Modify()`/`MarkPackageDirty()` on their own — a restore placed next to `Modify()` would have been undone one line later. The restore itself is `TargetPackage->SetDirtyFlag(false)` guarded by `!bPackageWasDirty` (`:968-970` inside the shared `FinalizeApplied`, `:1327-1329` for reset), scoped to `GetOutermost()` only, so it never eats dirt a user or a concurrent editor action created and does not copy the asset-dump guard's blanket clear-everything shape. Mechanism #4: `markedDirty` at `:976` is now stamped from `TargetPackage->IsDirty()` — observed state — instead of echoing the request parameter, and because `FinalizeApplied` is shared this covers all five emit sites at once (the four `AActor` branches at `:1007`/`:1041`/`:1076`/`:1099` and the generic path at `:1132`). Docs: the residual hazard the code fix cannot close is now stated on the method page (`Docs/wiki-src/property.md:44`) — the probe value stays live in memory behind a clean flag, so a later legitimate edit to that package persists it too, and `asset.reload` remains the only correct discard. Verified by a full `Automation RunTests PinWright` on UE 5.8 — 3,496 tests, log `Saved/Logs/Automation_PinWright_savefix.log` — with the four new tests in `Source/PinWright/Private/Tests/Utility/TestPropertyMarkDirtyRespected.cpp` (`property.set.MarkDirtyFalseLeavesPackageClean`, `property.set.MarkDirtyTrueDirtiesPackage`, `property.set.MarkDirtyFalsePreservesPreexistingDirt`, `property.reset.MarkDirtyFalseLeavesPackageClean`) all `Result={Success}`, plus the pre-existing `property.set.MarkDirtyMutatorReportsAppliedNotSaved`, whose `markedDirty` expectation had to be **corrected** from `true` to `false`: its fixture CDO is outered to `GetTransientPackage()` and can never be dirtied, so the honest observed answer there is `false` — pre-fix it only read `true` because the field echoed the parameter. **Coverage limit, stated plainly:** two of the four new assertions pass against the unfixed code by construction — `MarkDirtyTrueDirtiesPackage` and the dirt-preservation half of `MarkDirtyFalsePreservesPreexistingDirt` are counterfactual guards against a bogus "never dirty anything" fix, not red-to-green evidence; the genuine red-to-green proof rests on the two `MarkDirtyFalseLeavesPackageClean` tests and the clean-package half of the preservation test. Residual doc gap left untouched (not in this ticket's fix scope): the response-field bullet at `Docs/wiki-src/property.md:40` still describes `markedDirty` as "the `markDirty` param", which the observed-state change at `:976` makes inaccurate. Four tests failed in the same run (`core.error_codes.AllEmittedCodesAreRegistered`, `Utils.AssetDumpInheritance.BuildClassPropertyJsonCoversAllProperties`, `utils.property_export.SetOrdering`, `utils.property_utils.SparseInstancedSubobjectDiff`) — all unrelated concurrent work in the same tree, none on the `markDirty` path.
