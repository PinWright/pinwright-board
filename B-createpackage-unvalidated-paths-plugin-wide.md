---
id: B-createpackage-unvalidated-paths-plugin-wide
title: "61 direct CreatePackage call sites still compose their package path from caller-supplied text with no engine validation, so the double-slash Fatal that has already killed one editor is reachable from each of them"
status: OPEN
severity: Critical
category: bug
tags: [crash, editor-process-death, unvalidated-input, package-path, double-slash, createpackage, sweep, data-loss]
encounters: 1
lastSeen: 2026-08-30T17:00:00+05:00
---

# The class, not the instance

`CreatePackage` (`C:/UE_5.8/Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectGlobals.cpp:1086-1120`)
logs at **Fatal** for two inputs — a name containing `//` (`:1094-1096`) and a name that resolves to
empty (`:1118`). `Fatal` is not compiled out in any configuration and ends the PROCESS, so an
unvalidated caller string reaching it does not fail the call: it kills the editor and every unsaved
package in it, in editors that PinWright shares between agents. No `if (!Package)` check can catch
it, because nothing after the call is reached.

Two tickets have now fixed one instance each of this shape —
`B-foliage-add-type-name-with-slash-kills-the-editor` (**measured**: editor pid 7856, callstack
`CreatePackage() [UObjectGlobals.cpp:1099]` ← `FoliageHandler.cpp:1589` ←
`FRpcDispatcher::ProcessRequest()`) and
`B-landscape-create-grass-type-name-with-slash-kills-the-editor`. This ticket is the **remainder of
the sweep those two ran**: every other direct `CreatePackage` in the plugin whose path is composed
from caller text without an engine guard. It is filed rather than fixed because the sites span ~30
handler files that several agents were editing concurrently; a 30-file sweep landed mid-wave would
fight for every one of them.

## What was enumerated, and the engine facts that bound it

Every direct `CreatePackage(...)` under `Plugins/PinWright/Source/`, excluding `Tests/`:
**80 call sites across 37 files** (two further textual matches are a comment and an error-message
string, not calls).

Three engine facts were read in source before classifying anything:

- **`FString::operator/` does not double.** `PathAppend`
  (`Core/Private/Containers/String.cpp.inl:855-885`) pops the terminator and appends when the right
  side starts with `/`, so `"/Game/AI" / "/Game/X"` is `/Game/AI/Game/X`. It doubles only when the
  LEFT already ends with `/`. `FString::Printf(TEXT("%s/%s"), ...)` doubles unconditionally. **Both
  still fatal on a name that itself contains `//`** — `name: "a//b"` is a one-argument kill at every
  unguarded site regardless of which composition it uses.
- **AssetTools-routed creates are safe.** `IAssetTools::CreateAsset` sanitizes through
  `UPackageTools::SanitizePackageName` (`Editor/UnrealEd/Private/PackageTools.cpp:1523-1550`), which
  coalesces contiguous slashes. The hazard is confined to the plugin's own direct calls, which is
  why the enumeration is direct-call-only.
- **What counts as a guard.** `FPackageName::IsValidTextForLongPackageName`
  (`PackageName.cpp:1682-1714`) rejects `//`, a trailing slash, a missing leading slash, a too-short
  name and `INVALID_LONGPACKAGE_CHARACTERS`; `IsValidLongPackageName` adds the mount check.
  `IsValidAssetPath` (`Utils/PathUtils.cpp:152-159`) rejects `//`, `..` and `:`.
  `SanitizeProjectRelativePath` (`:34-89`) collapses `//` in a loop and rejects `..` and unmounted
  roots. `ValidateAssetCreationPath` (`:326-364`) chains the last two. A composed path passing any of
  these immediately before `CreatePackage` is safe.

**19 sites are safe or already fixed** and are listed in
`B-landscape-create-grass-type-name-with-slash-kills-the-editor`. The **61 below are not**.

## The 61

Paths relative to `Plugins/PinWright/Source/`, module `Private/` stem elided. Line numbers are HEAD
at filing time and several of these files are under concurrent edit, so **re-derive before fixing**.

### Worse than unguarded: the guard is computed and then discarded

These three call `SanitizeProjectRelativePath`, store the result, and then compose the package path
from the **raw** argument anyway. A reader auditing the function sees a sanitizer and moves on.

| site | computed | used instead |
|---|---|---|
| `Handlers/Level/LevelStructureHandler.cpp:221` | `SafeLevelPath` (`:184`) | `LevelPath / LevelName` (`:194`) |
| `Handlers/Level/LevelStructureHandler.cpp:1162` | `SafeAssetPath` (`:1146`) | `DataLayerAssetPath / DataLayerName` (`:1156`) |
| `Handlers/Level/LevelStructureHandler.cpp:1450` | `SafePath` (`:1434`) | `HlodLayerPath / HlodLayerName` (`:1444`) |

### Folder guarded, caller's NAME unchecked

The folder passes a sanitizer; the name is concatenated onto it raw, so `name: "a//b"` reaches the
Fatal through a guard that looks like it is working.

`Handlers/AI/AIHandler.cpp:248` · `Handlers/Animation/SkeletonHandler.cpp:654` ·
`Handlers/Editor/UtilityWidgetHandler.cpp:70` · `Handlers/UI/WidgetCreateHandler.cpp:64` ·
`Handlers/Physics/PhysicsHandler.cpp:212` (the only `Printf` one — `SavePath` is checked at `:174`,
`PhysicsAssetName` never is, `:187-188`)

### No guard at all

`Handlers/AI/AIHandler.cpp:763, 1145, 1640` (each `Path / Name` straight off `Ctx.GetString`, both
arguments raw) ·
`Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp:3131, 3221, 3462` ·
`Handlers/Animation/AnimationAuthoringHandler_BlendSpace.cpp:321, 418, 658` ·
`Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp:478, 1567, 2045` ·
`Handlers/Animation/PhysicsAssetHandler.cpp:331` ·
`Handlers/Audio/AudioAuthoringHandler.cpp:354, 443, 857, 919, 954, 1545, 1771, 2043, 2270, 2327, 2523, 2582, 2778`
(thirteen sites, one file — the largest single cluster) ·
`Handlers/Audio/MetaSound/MetaSoundPatchPresetHandler.cpp:78, 218` ·
`Handlers/Blueprint/BlueprintTypeDefinitionHandler.cpp:497` ·
`Handlers/Material/MaterialAuthoringHandler.cpp:1797, 2057, 3048, 3087, 3126, 3168, 4036, 4094` ·
`Handlers/Material/MaterialParameterCollectionHandler.cpp:217` ·
`Handlers/Material/TextureHandler.cpp:104, 2475, 2666` ·
`Handlers/Niagara/NiagaraHandler.cpp:571, 638` ·
`Handlers/Render/RenderHandler.cpp:281` ·
`Handlers/Sequencer/SequencerBakeHandler.cpp:620` ·
`Handlers/Systems/GASHandler.cpp:2458` ·
`Handlers/Systems/GameFrameworkHandler.cpp:98` ·
`Handlers/UI/WidgetAuthoringUtils.cpp:71` (a shared helper — its callers are the reachable surface) ·
`Utils/AssetUtils.cpp:1502, 1878` (shared helpers, same note) ·
`PinWrightChooser/.../ChooserAuthoringHandler.cpp:962` ·
`PinWrightPoseSearch/.../PoseSearchHandler.cpp:452, 575`

## Fix

The helper already exists and is already shared. `Handlers/PackagePathCompose.h` declares

```cpp
inline bool PinWrightComposeAssetPackagePath(const FString& FolderPath, const FString& AssetName,
                                             FString& OutPackagePath, FString& OutError);
```

which runs `FName::IsValidXName` + `INVALID_OBJECTNAME_CHARACTERS` on the bare name and
`FPackageName::IsValidLongPackageName(..., bIncludeReadOnlyRoots=true, &Reason)` on the composed
path, and surfaces both engine reason texts verbatim. Two files use it today. Converting a site is
three lines plus an `INVALID_ARGUMENT` refusal.

Sites whose input is one whole caller-supplied path rather than a folder+name pair want
`FPackageName::IsValidLongPackageName` directly, or `SanitizeProjectRelativePath`, not this helper.

**Sequence this by file, not by site**, one file per subagent, and re-derive line numbers first —
this list was taken while several of these files were being edited by other agents.

**Not taken: a central `CreatePackageChecked`.** A wrapper that rejected `//` and returned null
would make the class non-fatal without auditing anything, but it converts an editor death into a
silent null at 80 sites whose `if (!Package)` branches were written under the assumption that
CreatePackage does not fail — turning a loud defect into a quiet one. Validating at the argument,
where the caller can be told which rule it broke, is the shape the two fixed tickets took and the
one this ticket recommends.

## Cross-links

- `B-foliage-add-type-name-with-slash-kills-the-editor` — the measured editor death and the
  callstack. Read it for the mechanism; it is the only evidence any of this rests on.
- `B-landscape-create-grass-type-name-with-slash-kills-the-editor` — the second fix, the shared
  header, and the regression-test shape that asserts the refusal without ever driving the Fatal.
  Copy that test's construction: pair every bad name with a well-formed argument that makes a
  reverted build bail ABOVE the concatenation.

## Severity

**Critical.** Impact class is unchanged from the two fixed instances — process death plus loss of
unsaved asset and level data in a shared editor — and this ticket covers 61 doors into it rather
than one. **Reach bumped up and then back down:** the affected verbs collectively (audio asset
creation, material authoring, texture import, AI asset creation, widget creation) are hit far more
often than either fixed instance, which argues up; but each individual site needs a path-shaped or
`//`-bearing argument, which is not what a well-behaved caller sends, which argues down. Critical is
already the ceiling.

**What is measured and what is not.** Measured: the mechanism, once, on `foliage.add_type`, with a
callstack. Read from source and **not** executed: every site listed here, and the guard/no-guard
verdict on each. No site below was driven — confirming one costs an editor.

## History
- `#1-sweep-of-remaining-createpackage-sites` `OPEN` reporter — Produced by the whole-plugin sweep
  that `B-landscape-create-grass-type-name-with-slash-kills-the-editor` was asked to run: 80 direct
  `CreatePackage` sites enumerated, each traced back to the composition of its argument, 19 found
  guarded or already fixed and 61 not. The three engine facts that bound the classification
  (`PathAppend` does not double; `IAssetTools::CreateAsset` sanitizes; what counts as a guard) were
  each read in `C:/UE_5.8/Engine/Source/` at the line numbers quoted, not assumed. Nothing under
  `Plugins/PinWright/Source/` was edited for this ticket. Two sub-classes are called out because
  they read as safe and are not: three `LevelStructureHandler.cpp` sites that compute a sanitized
  path and compose from the raw one anyway, and five that guard the folder while leaving the
  caller's name unchecked. Dedup: the two fixed instances are cross-linked above and are excluded
  from the 61; no other board ticket covers `CreatePackage` path composition.
- `#2-level-structure-handler-three-sites` `OPEN` developer — Closed the three
  `Handlers/Level/LevelStructureHandler.cpp` sites (`create_level`, `create_data_layer`,
  `configure_hlod_layer`). **The "worse than unguarded" classification above is wrong and should
  not be carried into the remaining files:** all three assign the sanitizer's result back over the
  raw variable (`LevelPath = SafeLevelPath` at `:191`, `DataLayerAssetPath = SafeAssetPath` at
  `:1153`, `HlodLayerPath = SafePath` at `:1441` — all present at the filing commit, this is a
  mis-read of the reporter's, not a concurrent edit), so the FOLDER half was already guarded and
  the sanitized value was the right thing to compose from. These three belong in the "folder
  guarded, caller's NAME unchecked" bucket instead. Two real holes were found and closed: (a) the
  bare name is concatenated raw — `dataLayerName`/`hlodLayerName` had only an emptiness check, and
  `levelName`'s hand-rolled filter covers `/` and `\` but not `.`, so `".."` composes
  `"/Game/Maps/.."`, which CreatePackage trims to `"/Game/Maps/."` and `ResolveName2`
  (`UObjectGlobals.cpp:1219`) then empties, hitting the **second** Fatal at `:1118` from one
  argument; (b) the `if (!IsValidMountPoint(FullPath)) FullPath = TEXT("/Game/") + FullPath;`
  fallback below each composition prepends onto a path that already starts with `/` and so
  manufactures `"//"` itself — reachable for a folder on a mounted root other than
  `/Game|/Engine|/Script` plus a name carrying `\ * ? < >`, which are legal object-name characters.
  The shared `PinWrightComposeAssetPackagePath` was deliberately **not** used: it composes with
  `Printf("%s/%s")`, which doubles the separator when the folder ends in `/`, and
  `SanitizeProjectRelativePath` does not strip a trailing slash — so routing a routine
  `levelPath: "/Game/Maps/"` (which `FString::operator/` handles correctly) through it would start
  refusing valid input; and it never sees the string hole (b) produces. Its two engine rules are
  applied instead by two file-local statics, split across the points where each is meaningful:
  `PinWrightLevelStructureValidateBareName` (`FName::IsValidXName` + `INVALID_OBJECTNAME_CHARACTERS`,
  placed ABOVE each folder sanitizer) and `PinWrightLevelStructureValidatePackagePath`
  (`FPackageName::IsValidLongPackageName(..., bIncludeReadOnlyRoots=true)`, placed AFTER the
  mount-point fallback on the exact string handed to `CreatePackage`). Both surface the engine's
  reason text verbatim; refusals are `INVALID_ARGUMENT` (file uses raw literals, no `ErrorCodes::`
  adoption). Regression coverage: `Tests/World/TestLevelStructureNameSafety.cpp`, three new leaf
  ids `PinWright.level.structure.{create_level,create_data_layer,configure_hlod_layer}.NameCarryingAPathIsRefused`
  (`check_test_ids.py` CLEAN, 4833 ids). Fatal-unreachability: every bad name is paired with a
  `<verb>Path` of `"/Game/../../Engine/Content"`, which `SanitizeProjectRelativePath` rejects above
  every composition, so a build with the fix reverted answers `SECURITY_VIOLATION` (or, on
  `create_data_layer`, an even earlier world/WP/subsystem gate) and the test goes red on the wrong
  code while the process lives — no host-dependent fixture involved. A bare-name control asserts
  the refusal is not blanket. Not compiled and not run: the orchestrator builds after the wave.
