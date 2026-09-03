---
id: E-python-cannot-mark-package-dirty
title: "Package dirtying is unexposed to Python on UE 5.8, so 'dirty it yourself from python.execute' is not actionable advice"
status: DONE
severity: Medium
category: ergonomic
tags: [python, dirty-flag, save, docs, workaround]
encounters: 1
lastSeen: 2026-08-12T00:00:00Z
---

# Package dirtying is unreachable from Python in UE 5.8

`Actor.mark_package_dirty` and `Package.set_dirty_flag` are **both unexposed to Python on
UE 5.8**. This matters because the standard advice given when a PinWright verb mutates
state without the `Modify()` / `MarkPackageDirty()` ceremony — "just dirty the package
yourself from `python.execute`" — cannot be carried out. There is no Python-side
workaround for a missing dirty flag, only a force-save.

The route that does work:

```python
unreal.EditorAssetLibrary.save_asset('/Game/Maps/Dota2_Blockout', only_if_is_dirty=False)
```

Consequences:
- Every verb that mutates without dirtying is a genuine data-loss risk with no user-side
  mitigation short of an unconditional force-save of the whole package.
- Guidance and wiki text that suggests dirtying from Python must be corrected.

**Correction (see history `#2`):** "there is no Python-side workaround at all, only a
force-save" is wrong. `EditorAssetSubsystem.set_dirty_flag(obj, True)` works for content
assets (but refuses non-assets and any package containing a map), and
`SystemLibrary.transact_object(obj)` dirties anything via `UObject::Modify()` (but returns
void). What was genuinely missing: levels, non-asset objects, a by-path form, and a result
you can trust. The exact names `mark_package_dirty` / `set_dirty_flag` are indeed absent.

**Fix:** two parts.
1. Correct any doc/guidance that recommends Python-side dirtying; document the
   `save_asset(..., only_if_is_dirty=False)` force-save as the only Python route.
2. Consider exposing a small PinWright verb (e.g. `level.mark_dirty` / `asset.mark_dirty`)
   so scripts have a supported, targeted alternative to a force-save.

## History
- `#1-initial-observation` `OPEN` reporter — "Actor.mark_package_dirty and Package.set_dirty_flag are both unexposed to Python on UE 5.8, so the commonly-given 'dirty it from python.execute' workaround for missing Modify()/MarkPackageDirty() ceremony is not actionable. Only unreal.EditorAssetLibrary.save_asset(path, only_if_is_dirty=False) works. Docs must be corrected and a targeted mark-dirty verb considered."
- `#2-premise-partly-wrong-two-engine-routes-exist` `IN-REVIEW` developer — "Verified the premise against C:\UE_5.8 engine source. CONFIRMED: UObjectBaseUtility::MarkPackageDirty (UObjectBaseUtility.h:527), UPackage::SetDirtyFlag (Package.h:649), UPackage::IsDirty (Package.h:654) and UObject::Modify (Object.h:303) all carry NO UFUNCTION macro; PyGenUtil::IsScriptExposedFunction (PyGenUtil.cpp:1615) exports a function only when FUNC_BlueprintCallable|FUNC_BlueprintEvent is set, and meta=(ScriptMethod)/ScriptName are naming sugar that grant no exposure; UPackage is UCLASS(MinimalAPI) with zero UFUNCTIONs, so unreal.Package is a forced empty wrapper. CORRECTION to #1: the claim 'there is no Python-side workaround at all, only a force-save' is WRONG. Two exposed engine UFUNCTIONs reach the flag: (a) UEditorAssetSubsystem::SetDirtyFlag (EditorAssetSubsystem.h:233, impl :1162) — returns a real bool, but refuses !IsAsset() objects AND any package where ContainsMap() is true, so it can never dirty a level or an actor; (b) UKismetSystemLibrary::TransactObject (KismetSystemLibrary.h:2179, impl :3742) — forwards to UObject::Modify() so it dirties anything including maps, but returns VOID (refusal indistinguishable from success) and writes the object into the open transaction, corrupting undo when called after the edit. UEditorLoadingAndSavingUtils::GetDirtyMapPackages/GetDirtyContentPackages (FileHelpers.h:135,143) enumerate the dirty set read-only. Residual gap that justified new surface: levels + non-assets + by-path + an honest verified result."
- `#3-shipped-pinwright-package-library-and-asset-mark-dirty` `IN-REVIEW` developer — "Implemented. NEW: Source/PinWright/Private/Utils/PackageDirtyUtils.h/.cpp (namespace PinWright::PackageDirty — guards, package resolution honouring OFPA external packages, and the read-back verification; deliberately never forwards UObject::MarkPackageDirty()'s return value because it returns TRUE for a transient object and for an object with no package, UObjectBaseUtility.cpp:244,283); Source/PinWright/Public/PinWrightPackageLibrary.h + Private/PinWrightPackageLibrary.cpp (UBlueprintFunctionLibrary, 7 BlueprintCallable statics -> unreal.PinWrightPackageLibrary.{mark_package_dirty, mark_actor_package_dirty, mark_package_dirty_by_path, is_package_dirty, is_package_dirty_by_path, describe_mark_dirty_blocker, get_package_name}; no out-params by design, because PyGenUtil folds return+out into a truthy tuple and 'if lib.mark(...)' would then pass on failure); Source/PinWright/Private/Handlers/Asset/AssetMarkDirtyHandler.cpp (asset.mark_dirty + asset.is_dirty, sharing the same policy so the two surfaces cannot drift); Source/PinWright/Private/Tests/Core/TestPackageDirtyUtils.cpp (6 tests: clean->dirty asserted on UPackage::IsDirty not on the return value, already-dirty idempotence, null refusal through both the util and the reflected library, transient-object refusal with the transient package asserted still clean, five bad-path forms, package/object/subobject path folding). EDITED: Handlers/ErrorCodes.h (+ERR_MARK_DIRTY_REFUSED), Docs/error-code-catalog.md, Docs/wiki-src/python.md (new '## Marking packages dirty from Python' section stating the 5.8 gap with engine file:line evidence, the full signature table, a copy-pasteable snippet, and both engine alternatives), Docs/wiki-src/asset.md (### asset.mark_dirty / ### asset.is_dirty), Docs/wiki-src/safe-mutation-save.md. NOT BUILT — a separate integration agent owns the build; Python visibility of a UFUNCTION is a compile-and-run fact and is UNVERIFIED until the editor is back up. Runtime verification: unreal.PinWrightPackageLibrary.mark_package_dirty_by_path('/Game/Maps/X') then assert unreal.PinWrightPackageLibrary.is_package_dirty_by_path('/Game/Maps/X'). NOT done deliberately: no meta=(ScriptMethod) hoisting onto unreal.Object (would restore the literal actor.mark_package_dirty() spelling but adds compile risk and injects a name onto every UObject wrapper), and no dirty-package ENUMERATION verb — that is F-list-dirty-packages, and the engine's GetDirtyContentPackages/GetDirtyMapPackages already cover it from Python."
- `#4-runtime-verified-and-committed` `DONE` tester — Built and runtime-verified.
  **STEP 0 (the compile-and-run fact nobody could prove) PASSES:** `hasattr(unreal,
  "PinWrightPackageLibrary")` is `True` and all seven wrappers are present as
  `unreal.PinWrightPackageLibrary.*` — `mark_package_dirty`, `mark_actor_package_dirty`,
  `mark_package_dirty_by_path`, `is_package_dirty`, `is_package_dirty_by_path`,
  `describe_mark_dirty_blocker`, `get_package_name`. **The `MarkPackageDirty` name-hiding risk did not
  materialise**: the static did not collide with the inherited non-virtual
  `UObjectBaseUtility::MarkPackageDirty()` in either MSVC or UHT, so no rename to `MarkDirty` was
  needed. **Independent engine cross-check passed:** after `mark_package_dirty_by_path`, the package
  appears in `unreal.EditorLoadingAndSavingUtils.get_dirty_map_packages()` — engine code entirely
  separate from `PinWright::PackageDirty`. `mark_actor_package_dirty` + `is_package_dirty` on a live
  actor agree. **Refusals are False + printable, never a crash or silent True:** `None`, `""`,
  `"not a path"`, and a well-formed-but-unloaded `/Game/Definitely/Not/Loaded_zzz` all return `False`;
  `describe_mark_dirty_blocker(None)` returns `"object is null"`. **RPC surface:** `asset.is_dirty` on
  a clean loaded package returns `isDirty:false`; `asset.mark_dirty` on it returns
  `wasDirty:false, isDirty:true, saved:false` (clean→dirty transition proven); `asset.mark_dirty` on
  an unloaded path returns `PACKAGE_NOT_FOUND`. All 6 `PinWright.core.package_dirty.*` tests pass —
  including the `/Temp/` scratch-package tests 1/2/6 that were flagged as possible harness failures, so
  that concern did not materialise either — plus the 2 wiki doc-contract tests
  (`MethodPage.AssetMarkDirty`, `NamespacePage.PythonMarkPackageDirty`). Every flag set during
  verification was cleared afterwards; the project ends with zero dirty packages. Committed as `498929c6`.
