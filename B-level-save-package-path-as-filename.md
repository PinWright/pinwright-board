---
id: B-level-save-package-path-as-filename
title: "Every level-save verb passes a long package name where FEditorFileUtils::SaveLevel expects a filesystem filename, so the engine writes the .umap to an extensionless drive-relative path (C:\\Game\\...) instead of the project, clears the dirty flag, and silently discards the map's edits"
status: IN-REVIEW
severity: Critical
category: bug
tags: [level-save, path-conversion, silent-data-loss, save, umap, customer-report]
encounters: 1
lastSeen: 2026-08-09T04:36:57Z
---

# Every level-save verb passes a long package name where FEditorFileUtils::SaveLevel expects a filesystem filename, so the engine writes the .umap outside the project and silently discards the map's edits

External QA report, UE 5.7.4, Windows, PinWright running in-editor. The reporter called `level.save` on an already-saved map (`/Game/MOTOR/Maps/LV_MOTOR_TestArena`), got `saved: true`, and then found the project's `.umap` byte-for-byte unchanged. The map's edits were gone from the project and the package's dirty flag had been cleared, so Save-All-on-exit never prompted and the work was lost on editor close.

## What's wrong

`McpSafeLevelSave` hands `FEditorFileUtils::SaveLevel` a long package name (`/Game/MOTOR/Maps/LV_MOTOR_TestArena`) in the parameter the engine declares as `DefaultFilename` — a *filesystem* filename. The engine forwards it as `ForceFilename`, overriding the level's real on-disk path, and rebuilds a destination that has no `.umap` extension and no drive letter. `FPaths::IsRelative` treats the leading `/` as rooted (`Runtime/Core/Private/Misc/Paths.cpp:1267-1272`: `InPath[0] == '/'` → "Root of the current directory on Windows"), so Windows resolves it against the current drive and the package lands at **`C:\Game\MOTOR\Maps\LV_MOTOR_TestArena`, extensionless**. The write genuinely succeeds — at the wrong path — so the dirty flag is cleared and nothing anywhere reports a problem.

## Root cause

`Source/PinWright/Private/Utils/AssetUtils.cpp:510-543`, function `McpSafeLevelSave`. It normalizes `FullPath` into a long package name at `:518-528` (mount-point prefix, extension strip) and then calls, at `:543`:

```cpp
bSaveSucceeded = FEditorFileUtils::SaveLevel(Level, *PackagePath);
```

The engine's second parameter is `DefaultFilename` (declared `Engine/Source/Editor/UnrealEd/Public/FileHelpers.h:304` on 5.7: `static UNREALED_API bool SaveLevel(ULevel* Level, const FString& DefaultFilename = TEXT( "" ), FString* OutSavedFilename = nullptr );`). `SaveLevel` (`FileHelpers.cpp:4242`) forwards it **unconditionally as `ForceFilename` whenever it is non-empty** (`FileHelpers.cpp:4288-4296`, the ternary at `:4289`):

```cpp
bLevelWasSaved = SaveWorld( WorldToSave,
                            DefaultFilename.Len() > 0 ? &DefaultFilename : NULL,
                            ...
```

The `Filename = GetFilename(Level->OwningWorld)` lookup at `:4255-4259` that would have produced the level's *real* on-disk path feeds only the has-this-been-saved-before branch; it never reaches the `SaveWorld` call, so the level's true path is overridden even for a map that has been saved a hundred times.

`SaveWorld` then splits `ForceFilename` (`FileHelpers.cpp:944-948`) into `Path = FPaths::GetPath(*ForceFilename)` = `/Game/MOTOR/Maps` and `CleanFilename = FPaths::GetCleanFilename(*ForceFilename)` = `LV_MOTOR_TestArena`, and reassembles them at `:979-996` into `FinalFilename = "/Game/MOTOR/Maps/LV_MOTOR_TestArena"` — with **no `.umap` extension**, because the extension was never there to split off.

The "not within the game or engine content folders" guard at `:999-1003` does not fire. `FPackageName::TryConvertFilenameToLongPackageName` (`Runtime/CoreUObject/Private/Misc/PackageName.cpp:819`) passes the string through `InternalFilenameToLongPackageName`, finds no dot, backslash or colon, and returns it verbatim as an already-valid long package name (`:837-848`). The round trip is a no-op, so the guard sees a legal package name and lets the bad `FinalFilename` through.

Finally `FileHelpers.cpp:1210-1213` execs the save with that string in the `FILE=` slot:

```cpp
bSuccess = GEditor->Exec(
    nullptr,
    *FString::Printf(TEXT("OBJ SAVEPACKAGE PACKAGE=\"%s\" FILE=\"%s\" SILENT=true AUTOSAVING=%s KEEPDIRTY=%s"), *Package->GetName(), *FinalFilename, *AutoSavingString, *KeepDirtyString),
    SaveOutput.GetOutputDevice());
```

`UEditorEngine::Exec_Obj` parses `FILE=` into `TempFname` (`EditorServer.cpp:4637`) and passes it straight to `UEditorEngine::SavePackage` (`:4673`) with no path validation. The reporter's `FILE=` hypothesis is correct; the interpolation happens entirely engine-side, driven by the plugin's argument.

## Why the dirty flag clears

`UPackage::SavePackage` genuinely succeeded — at the wrong path — so `SavePackage2.cpp:3558-3560` clears the flag on success when `KEEPDIRTY=false`, which is what `SaveWorld` passes for a non-PIE save (`bKeepDirty = bPIESaving`, `EditorServer.cpp:4652` parses it into `SAVE_KeepDirty`):

```cpp
if (!SaveContext.IsKeepDirty())
{
    SaveContext.GetPackage()->SetDirtyFlag(false);
```

Nothing in the plugin touches the dirty flag. This is why Save-All-on-exit does not prompt and the work is lost on editor close.

## Evidence

The orphan tree exists on this dev machine and is entirely composed of real package data. `C:\Game\` holds **824 files, 25 MB (25,435,963 bytes)**. Every one of the 824 files begins with `c1 83 2a 9e` = `PACKAGE_FILE_TAG` — checked byte-by-byte across the whole tree, not sampled — i.e. genuine serialized package bytes with no extension. `C:\Game\Maps\SavedLevel` starts `c1 83 2a 9e f7 ff ff ff`. **786 of the 824** are `__EARG_LightingLevelHonestyProbe_*` packages emitted by the lighting probe path. The oldest file is `C:\Game\Maps\TestLitLevel`, dated **2026-02-28**, which is the earliest known onset — over five months of writes.

The remaining 38 files are the smoking gun: `C:\Game\Maps\` contains `FuzzReplay_WP_A`, `FuzzReplay_Flat_B`, `OracleWPReplay_Z1`, `OracleFlatReplay_Z3`, `OracleColdReplay_WP1`, `OracleColdReplay_WP2`, `OpenWorldSandbox`, `OpenWorldPrototype`, `LightingSandbox`, `LightingSandbox_v2`, `PW_DiskTest_LS`, `LS_Tmp`, `Prototype\OW_Prototype`, and `System\FrontEnd\Maps\L_Core`. Those are the exact map names from `B-create-level-saved-true-no-umap` `#1`/`#3`/`#4`/`#5`/`#6` and from `B-level-save-no-completion-signal` `#3`. Those history entries state that a recursive search "found nothing — no file landed at the resolved path **nor anywhere else**"; the search was scoped to `Content/`, and the packages were in fact written, to `C:\Game\`. This ticket is the real root cause those investigations were reaching for: the save was never a no-op, it was a misdirected write.

Reporter's independent evidence: `git hash-object` on `Content/.../LV_MOTOR_TestArena.umap` after a "successful" `level.save` matched the blob in HEAD — the project file was byte-for-byte unchanged. Working workaround: `editor.save_all`.

## Affected call sites

- `Source/PinWright/Private/Utils/AssetUtils.cpp:543` — `FEditorFileUtils::SaveLevel(Level, *PackagePath)`. The shared root; feeds everything below.
- `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:544` — `FEditorFileUtils::SaveMap(NewWorld, SavePath)` in `level.create`. A **second independent instance** of the same mistake, and it has no disk gate at all, so it returns plain `SendSuccess` at `:551`. Two lines above, at `:537-542`, the correctly-converted `Filename` is already computed by `TryConvertLongPackageNameToFilename` — and is used only to `MakeDirectory` the destination, then discarded; the save call gets `SavePath` instead.
- `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:317` — `level.save`, the reported verb.
- `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:421` — `level.save_as`.
- `Source/PinWright/Private/Handlers/Level/LevelStructureHandler.cpp:286` — `level.structure.create_level`.
- `Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp:908` — `lighting.create_lighting_enabled_level`, the lighting level probe, source of 786 of the 824 orphan files.

## Not affected

Scoped deliberately so the fix is not over-broad. No `FILE=`, `SAVEPACKAGE`, `OBJ SAVE` or `MAP SAVE` string is constructed anywhere in the plugin — a grep over `Source/PinWright/Private/` returns zero hits. The interpolation is entirely engine-side. All asset saves are clean: `SaveLoadedAssetThrottled` (`AssetUtils.cpp:652-712`) ends at `UEditorAssetLibrary::SaveLoadedAsset(Asset, ...)` which takes a `UObject*`, and `McpSafeAssetSave` (`:220`), `SavePackageHelperAI` (`Handlers/AI/AIHandler.cpp:162-167`) and `SavePackageHelperChar` all delegate into it — none of them ever forms a filename. `level.export` (`LevelHandler.cpp:858-863`) is clean: it explicitly resolves a relative `ExportPath` against `FPaths::ProjectDir()` before exporting, with a comment naming the engine-CWD trap this ticket is about.

## Reference implementation

`editor.save_all` is correct and is the model to copy: `Handlers/Editor/EditorCommandHandler.cpp:404-426` → `SaveDirtyPackagesWithIntegrityGate` (`:235-281`) → `UEditorAssetLibrary::SaveAsset(PackagePath, false)` (`:267`) → `UEditorAssetSubsystem::SaveAsset` → `UEditorLoadingAndSavingUtils::SavePackages`, which takes `UPackage*` and never a filename, so the engine derives the path from the package itself. `editor.quit` (`Handlers/Editor/EditorQuitHandler.cpp:190`, `FEditorFileUtils::SaveDirtyPackages`) is likewise correct. Both are why `editor.save_all` is the working workaround.

## Fix

1. In `McpSafeLevelSave`, convert before calling and use the out-param to prove where the bytes landed:

```cpp
FString MapFilename;
if (!FPackageName::TryConvertLongPackageNameToFilename(
        PackagePath, MapFilename, FPackageName::GetMapPackageExtension()))
{
    // '<path>' does not resolve to a mounted map filename — fail loudly
    return false;
}
FString ActuallySavedTo;
bSaveSucceeded = FEditorFileUtils::SaveLevel(Level, MapFilename, &ActuallySavedTo);
```

then require, in the success predicate, that `ActuallySavedTo` is non-empty and `FPaths::IsSamePath` with the full-path form of `MapFilename`. Reuse `MapFilename` in the existing verification block at `:551-557` instead of re-converting. This kills the whole class at the root, not just this instance — every verb in the affected-call-sites list except `level.create` routes through here.

2. `LevelHandler.cpp:544` — pass the already-computed `Filename` instead of `SavePath`, and fail if the conversion at `:537-539` failed (today its failure is silently ignored: only the `MakeDirectory` is skipped).

3. Delete the orphan `C:\Game\` tree once the fix ships (25 MB, 824 files).

## Version note

No `#if` guard needed. `FEditorFileUtils::SaveLevel(ULevel*, const FString&, FString*)` has an identical signature and identical `ForceFilename` forwarding on 5.3 (`FileHelpers.cpp:3860`), 5.4 (`:3884`), 5.5 (`:3901`), 5.6 (`:3970`), 5.7 (`:4242`) and 5.8 (`:4240`), verified against all six local engine installs. The `OBJ SAVEPACKAGE PACKAGE="%s" FILE="%s" SILENT=true AUTOSAVING=%s KEEPDIRTY=%s` format string is byte-identical on all six (only the surrounding `GEditor->Exec` call is reformatted across lines on 5.7/5.8 versus one line on 5.3-5.6). So this bug is present on every supported engine version.

## Test gap

`PinWright.level.save.ValidParamsNoCrash` (`Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp:219-228`) asserts only `IsRegistered(TEXT("level.save"))` and never invokes the handler; `PinWright.level.save_as.ValidParamsNoCrash` (`:244-253`) is the same shape. The regression tests in `Source/PinWright/Private/Tests/Core/TestLevelSaveLoadUtils.cpp:335-430` exercise the *predicates* with synthetic booleans against a never-saved `CreatePackage`'d package, and the file-writing path is never run — as the test's own scope comment at `:329-334` concedes: "It does NOT invoke the handlers themselves". A real test must, on an **already-saved** map: capture `IFileManager::GetTimeStamp` + `FileSize` of the `.umap` before `level.save` and assert both changed; assert `SaveLevel`'s `OutSavedFilename` resolves under `FPaths::ProjectContentDir()`; and assert no file was created outside the project.

## Distinct from

`B-level-save-verify-existence-not-freshness` covers the *masking* defect — the existence-only disk gate that is why this went undetected for five months. This ticket covers the wrong-path write itself. Both must be fixed; fixing either alone leaves a hole. Also distinct from `B-level-save-saved-true-in-memory-no-umap` and `B-create-level-saved-true-no-umap` (both IN-REVIEW), which correctly diagnosed the *symptom* (a save reporting success with no `.umap` at the expected path) but attributed it to the engine declining to write for an in-memory-only or inactive world; the `C:\Game\Maps\` contents named in Evidence show the engine did write, just elsewhere. Their fixes made the reports honest and remain correct; this ticket makes the write land.

## Severity

Critical. Impact class is "a write that corrupts or loses asset data": the map's edits are silently discarded from the project while the dirty flag is cleared, so Save-All-on-exit never prompts and the loss is unrecoverable at editor close. Reach pushes it up rather than down — `level.save` is a core every-session verb, and the 824-file orphan tree dating to 2026-02-28 shows the loss has been continuous, not incidental. There is no in-band signal of any kind: the RPC reports success, the editor reports no error, and the only trace is a directory outside the project that nobody looks at.

## History
- `#1-filed-from-customer-report` `OPEN` reporter — External QA report (UE 5.7.4, Windows, PinWright in-editor): `level.save` on `/Game/MOTOR/Maps/LV_MOTOR_TestArena` returned `saved:true` while `git hash-object` on the project's `.umap` matched HEAD byte-for-byte, and the dirty flag was cleared so Save-All-on-exit never prompted; `editor.save_all` works. Root-caused to `Utils/AssetUtils.cpp:543` passing a long package name into `FEditorFileUtils::SaveLevel`'s `DefaultFilename` slot (a filesystem filename, `FileHelpers.h:304`), which the engine forwards unconditionally as `ForceFilename` (`FileHelpers.cpp:4288-4296`) overriding the level's real path, splits and rebuilds into an extensionless `FinalFilename` (`:944-948`, `:979-996`), passes the content-folder guard because `TryConvertFilenameToLongPackageName` round-trips a `/Game/...` input unchanged (`Misc/PackageName.cpp:837-848`), and execs into `OBJ SAVEPACKAGE ... FILE="/Game/MOTOR/Maps/LV_MOTOR_TestArena"` (`FileHelpers.cpp:1210-1213` → `EditorServer.cpp:4637-4673`), where the leading `/` is rooted per `FPaths::IsRelative` (`Paths.cpp:1267-1272`) so Windows resolves it drive-relative to `C:\Game\...`; the dirty flag then clears legitimately at `SavePackage2.cpp:3558-3560` because the write really did succeed. Confirmed the reporter's `FILE=` hypothesis: the interpolation is engine-side, driven by the plugin's argument. Measured the orphan tree on this machine — `C:\Game\` holds 824 files / 25 MB / 25,435,963 bytes, every single one starting with `PACKAGE_FILE_TAG` (`c1 83 2a 9e`), 786 of them `__EARG_LightingLevelHonestyProbe_*` from the lighting probe, oldest `C:\Game\Maps\TestLitLevel` dated 2026-02-28 as the earliest known onset. `C:\Game\Maps\` also contains `FuzzReplay_WP_A`, `FuzzReplay_Flat_B`, `OracleWPReplay_Z1`, `OracleFlatReplay_Z3`, `OracleColdReplay_WP1/WP2`, `OpenWorldSandbox`, `OpenWorldPrototype`, `LightingSandbox`, `PW_DiskTest_LS`, `LS_Tmp`, `Prototype\OW_Prototype` and `System\FrontEnd\Maps\L_Core` — the exact maps that `B-create-level-saved-true-no-umap` and `B-level-save-no-completion-signal` recorded as "no file landed anywhere", refuting that conclusion (their searches were scoped to `Content/`) and identifying this as the root cause behind those symptoms. Six affected call sites all inherit the shared root (`AssetUtils.cpp:543`), plus a second independent instance at `LevelHandler.cpp:544` where `level.create` passes `SavePath` to `SaveMap` while the correctly-converted `Filename` computed at `:537-542` is used only for `MakeDirectory` and then discarded. Scoped what is NOT affected: zero `FILE=`/`SAVEPACKAGE` string construction in the plugin, all asset saves route through `UEditorAssetLibrary::SaveLoadedAsset`/`SaveAsset` taking a `UObject*`/package path, and `level.export` correctly resolves against `FPaths::ProjectDir()`. Verified the identical `SaveLevel` signature and `ForceFilename` forwarding on all six local engine installs (5.3 `:3860`, 5.4 `:3884`, 5.5 `:3901`, 5.6 `:3970`, 5.7 `:4242`, 5.8 `:4240`) with a byte-identical exec format string, so no version guard is needed and every supported engine is affected. Dedup: `B-level-save-package-path-as-filename` did not exist; scanned the board for `level-save`/`savelevel`/`save-map`/`package-path`/`filename`/`umap` near-duplicates and found only the symptom-level tickets named in "Distinct from", none of which identifies the filename-vs-package-name argument defect.
- `#2-filename-not-package-path` `IN-REVIEW` developer — Fixed both instances. `Source/PinWright/Private/Utils/AssetUtils.cpp`, `McpSafeLevelSave` (`:510`): the long package name is resolved to a real filesystem path through the file's own existing `ResolveLevelPackageToMapFilename` helper (`:376`) at `:538`, and an unresolvable mount now returns `false` with an error log instead of handing the `/Game/...` string to `FEditorFileUtils::SaveLevel`'s `DefaultFilename` slot; the save at `:586` is the 3-arg overload `SaveLevel(Level, MapFilename, &SavedFilename)` and the engine-reported destination is compared against the expected path at `:609-615` (`FPaths::IsSamePath` on `FPaths::ConvertRelativePathToFull` forms) as a positive destination proof, gating success at `:624` — an **empty** out-param is deliberately treated as "unknown, assume as requested" so an engine build that does not populate it cannot regress every save into a false failure. A destination that resolves under the `/Temp` mount (a never-saved `/Temp/Untitled_N` world) is now refused outright at `:550-556`: pre-fix that case died on the conversion, and letting the now-working conversion through would have started silently persisting into `Saved/Temp` where the content browser can never see it, so the refusal keeps the existing "use `level.save_as` with a `/Game/...` path" guidance and states it in the log. The second, independent instance is fixed in `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp` `level.create` (`:534`): the correctly-converted `Filename` computed at `:540` is now what reaches `FEditorFileUtils::SaveMap` at `:553` (it was previously computed, used only for `MakeDirectory`, then discarded while `SavePath` went to `SaveMap`), and a failed conversion returns `INVALID_PATH` at `:543` rather than being silently ignored. Verified by a full `Automation RunTests PinWright` on UE 5.8 — 3,496 tests completed, log `Saved/Logs/Automation_PinWright_savefix.log` — with the three new regression tests in `Source/PinWright/Private/Tests/World/TestLevelSavePathTargeting.cpp` all `Result={Success}`: `PinWright.level.save.WritesToProjectContentNotDriveRoot`, `PinWright.level.save.RewritesExistingUmap`, `PinWright.level.create.VerifiesUmapOnDisk`. The strongest end-to-end evidence is external to the assertions: the orphan tree at `C:\Game` held **824 files before the suite run and 824 after** — a full suite exercising every level-save verb wrote nothing outside the project, where previously those same code paths are exactly what filled it. Four tests failed in that run (`PinWright.core.error_codes.AllEmittedCodesAreRegistered`, `PinWright.Utils.AssetDumpInheritance.BuildClassPropertyJsonCoversAllProperties`, `PinWright.utils.property_export.SetOrdering`, `PinWright.utils.property_utils.SparseInstancedSubobjectDiff`); none touches level save — they belong to unrelated concurrent work in the same working tree (the error-code failure names `ASSET_COMPILING`/`PIE_STOP_FAILED`, both since added to `Handlers/ErrorCodes.h` by that other work). Fix step 3 (deleting the 25 MB / 824-file `C:\Game` tree) is deliberately **not** done: the count is the load-bearing evidence above and should be deleted only after the tester has re-checked it.
