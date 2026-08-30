---
id: B-landscape-create-grass-type-name-with-slash-kills-the-editor
title: "landscape.create_grass_type concatenates its `name` onto a hardcoded /Game/Landscape with no validation, so a name carrying a path — the only thing a caller can try, because the verb exposes no savePath — builds `/Game/Landscape//Game/...` and CreatePackage's Fatal log kills the editor process outright"
status: OPEN
severity: Critical
category: bug
tags: [landscape, create_grass_type, crash, editor-process-death, unvalidated-input, package-path, double-slash, data-loss, shared-editor]
encounters: 1
lastSeen: 2026-08-30T17:00:00+05:00
---

# The fifth instance of one defect class, in a second namespace

`landscape.create_grass_type` builds its package path by blind `Printf` inside its
`AsyncTask(GameThread)` body and hands the result straight to `CreatePackage`.
`Handlers/Environment/LandscapeHandler.cpp:1675-1689` (pre-fix line numbers):

```cpp
    FString PackagePath = TEXT("/Game/Landscape");
    FString AssetName = Name;
    FString FullPackagePath = FString::Printf(TEXT("%s/%s"), *PackagePath, *AssetName);
    ...
    UPackage *Package = CreatePackage(*FullPackagePath);
```

`Name` is never checked — it is read at `:1636-1640` (`TryGetStringField` + an `IsEmpty()` test) and
carried into the lambda unmodified. A `name` beginning with `/` yields `/Game/Landscape//Game/...`,
and `CreatePackage` treats a double slash as unrecoverable —
`C:/UE_5.8/Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectGlobals.cpp:1094-1097`:

```cpp
		if( InName.ToView().Contains(TEXT("//")))
		{
			UE_LOGF(LogUObjectGlobals, Fatal, "Attempted to create a package with name containing double slashes. PackageName: %ls", PackageName);
		}
```

`Fatal` is not compiled out in any configuration. There is a second `Fatal` at `:1118` for a name
that resolves to empty. Neither is catchable: the handler's `if (!GrassType)` at `:1691` is never
reached, because the process is gone before `CreatePackage` returns.

## Not measured here — and it does not need to be

**This is the same mechanism the sibling ticket measured, in a different file.**
`B-foliage-add-type-name-with-slash-kills-the-editor` recorded the death of a live editor (pid 7856,
`/Game/Maps/PW_VegetationTest`) from one `foliage.add_type` call, with the callstack naming
`CreatePackage() [UObjectGlobals.cpp:1099]` ← the plugin handler line ←
`FRpcDispatcher::ProcessRequest()`. That ticket's own `#2` history entry closes by naming this site
as the fifth instance found and deliberately left unedited because it belongs to another namespace.

Reproducing it here means killing a second editor to confirm a mechanism already proven with a
callstack. **Not executed, deliberately.** Everything below is derived from source at the line
numbers quoted, each re-read before the fix.

## Why a caller reaches this, rather than it being an exotic input

**`landscape.create_grass_type` had no path parameter of any kind.** Its registered params
(`:1622-1628` pre-fix) were `name`, `meshPath`, `density`, `minScale`, `maxScale`, and the
destination `/Game/Landscape` was a literal at `:1675`. `meshPath` — the *other* path argument in
the same handler — is sanitized eight lines above (`:1647-1653`,
`SanitizeProjectRelativePath` → `SECURITY_VIOLATION`), so the file already owns the habit; `name` is
simply the argument it is not applied to. A caller who wants the grass type somewhere other than
`/Game/Landscape` has exactly one lever to try, and pulling it ends the process. The wiki page for
the verb documented neither the hardcoded destination nor any constraint on `name`.

## The whole-plugin sweep, which is the reason this ticket is not the end of it

Every direct `CreatePackage(...)` under `Plugins/PinWright/Source/` was enumerated and each one's
path traced back to its composition. **81 direct call sites** across 37 files.

Two engine facts bound the hazard and were verified in engine source before classifying anything:

- `FString::operator/` (`PathAppend`, `Core/Private/Containers/String.cpp.inl:855-885`) does **not**
  produce `//` when the right side starts with `/` — it pops and appends, yielding a single slash.
  It only doubles when the LEFT already ends with `/`. So `Folder / Name` is a weaker hazard than
  `Printf("%s/%s")`, which doubles unconditionally. Both still fatal on a `Name` containing `//`.
- `IAssetTools::CreateAsset` routes through `UPackageTools::SanitizePackageName`
  (`Editor/UnrealEd/Private/PackageTools.cpp:1523-1550`), which coalesces contiguous slashes. Every
  AssetTools-routed create is therefore **safe**, and the hazard is confined to the plugin's own
  direct `CreatePackage` calls.
- `FPackageName::IsValidTextForLongPackageName` (`PackageName.cpp:1682-1714`) rejects `//`, a
  trailing slash, a missing leading slash, a too-short name and `INVALID_LONGPACKAGE_CHARACTERS`.
  `IsValidAssetPath` (`Utils/PathUtils.cpp:152-159`) and `SanitizeProjectRelativePath`
  (`:34-89`, which collapses `//` in a loop) do the same job. A composed path passing any of these
  before `CreatePackage` is safe.

**18 sites are safe** — their composed path passes one of those guards: `PwMusicGraph.cpp:590`,
`AIHandler.cpp:2333`, `BehaviorTreeHandler.cpp:583`, `EQSHandler.cpp:377`,
`AnimSequenceCreate.cpp:933`, `CharacterHandler.cpp:201`, `MaterialAuthoringHandler.cpp:698`,
`GASHandler.cpp:772`, `WorldPartitionHandler.cpp:156`, `FoliageHandler.cpp:194/1826/2378/2499`
(the sibling ticket's fix), `PCGGraphCreate.cpp:104`, `ChaosVehicleHandler.cpp:246`,
`MGIRCompiler.cpp:344/385`, `PwSkelAssetCreate.cpp:872`.

**63 sites are not**, of which this ticket fixes one. The remaining **62 are filed separately** as
`B-createpackage-unvalidated-paths-plugin-wide`, because they span ~30 handler files that other
agents are editing concurrently and cannot be swept safely inside one ticket's diff. Two findings
from that sweep are worth naming here because they are worse than "no guard":

- `LevelStructureHandler.cpp:221`, `:1162`, `:1450` each **compute** a sanitized path
  (`SafeLevelPath` / `SafeAssetPath` / `SafePath`) and then compose the package path from the
  **raw, unsanitized** argument anyway. The guard is present, written, and discarded.
- Eight sites (`AIHandler.cpp:248`, `SkeletonHandler.cpp:654`, `UtilityWidgetHandler.cpp:70`,
  `WidgetCreateHandler.cpp:64`, `PhysicsHandler.cpp:212`, plus the three above) sanitize the FOLDER
  and leave the caller's NAME unchecked — a `name` of `a//b` reaches the Fatal through a guard that
  looks like it is doing the job.

## Fix

Route the composition through the shared helper the sibling ticket's fix introduced, and give the
verb the `savePath` its neighbours already have. See History `#2`.

## Cross-links

- `B-foliage-add-type-name-with-slash-kills-the-editor` — **the sibling that established the
  mechanism.** Same defect, same engine `Fatal`, and the only one of the two with a MEASURED editor
  death and a callstack. Its fix wrote the validator this ticket reuses; read it first.
- `B-createpackage-unvalidated-paths-plugin-wide` — the other 62 sites this ticket's sweep found and
  did not touch. This ticket closing does not close that class.
- `B-create-anim-blueprint-duplicate-name-crash` — the nearest sibling in kind: a caller string
  reaching an engine fatal through a creation verb.

## Severity

**Critical.** Impact class: process death, and with it loss of every unsaved package in a shared
editor — the band the rubric puts at Critical, reached by one well-formed argument to a registered
verb on its documented happy path. **Reach modifier declined in both directions.** No upward bump:
`create_grass_type` is not an every-session verb. No downward "rare edge path" bump either: a
path-shaped `name` is what the surrounding namespace's conventions train a caller to supply, and
before this fix the verb offered no other way to choose a destination.

## History
- `#1-name-concatenation-fatals-createpackage` `OPEN` reporter — Filed from the sibling ticket's
  closing note, then re-derived from source at HEAD before any edit: unvalidated `name` read at
  `LandscapeHandler.cpp:1636-1640`, blind concatenation `:1675-1677`, `CreatePackage` `:1689`,
  registered params `:1622-1628` (no path parameter of any kind), engine `Fatal`
  `UObjectGlobals.cpp:1094-1097` with a second at `:1118`. The file already applies
  `SanitizeProjectRelativePath` to its OTHER path argument (`meshPath`, `:1647-1653`), so this is
  an unapplied validator rather than a missing one. **Deliberately NOT reproduced** — the mechanism
  is proven by the sibling's callstack and confirming it here costs a second editor.
  Whole-plugin sweep run as part of this report: 81 direct `CreatePackage` sites, 18 guarded, 63
  not. Dedup: no board ticket covers `create_grass_type`'s `name`; the three neighbours under
  Cross-links were each read and are different defects.
