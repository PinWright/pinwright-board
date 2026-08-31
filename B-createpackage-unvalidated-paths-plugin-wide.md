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
- `#3-animation-authoring-nine-sites` `OPEN` developer — Closed the nine `animation.authoring`
  sites, all in the "no guard at all" class, all one idiom (`Path / Name` off `Ctx.GetString` with
  `Path` pre-normalized by `AnimationAuthoringHelpers::NormalizeAnimPath`, composed inline at the
  `CreatePackage` line). Re-derived line numbers matched the ticket exactly at pick time and moved
  during the edits; the sites by verb are
  `AnimationAuthoringHandler_Sequence.cpp` `create_animation_sequence` / `create_montage` /
  `create_composite`, `AnimationAuthoringHandler_BlendSpace.cpp` `create_blend_space_1d` /
  `create_blend_space_2d` / `create_aim_offset`, and
  `AnimationAuthoringHandler_AnimBlueprint.cpp` `create_pose_library` / `create_ik_retargeter` /
  the `#elif MCP_HAS_CONTROLRIG_BLUEPRINT` (UE 5.1-5.4) fallback branch of `create_control_rig`.
  **No shared local composition helper existed** to guard once: the three files share
  `AnimationAuthoringHelpers` but it carries only `NormalizeAnimPath` (folder normalization), and
  each site composed inline. A per-cluster `Ctx`-aware wrapper was considered and rejected — it
  would save ~2 lines a site, pull `FHandlerContext` into a pure-data helpers header, and diverge
  from the shape `FoliageHandler.cpp` / `LandscapeHandler.cpp` already ship. All nine therefore call
  the plugin-wide `PinWrightComposeAssetPackagePath` directly, refusing `INVALID_ARGUMENT` with the
  engine reason text quoted. **Error-code adoption differs per file and was checked independently:**
  `_Sequence.cpp` already cites `ErrorCodes::` 69 times with zero raw literals, so it uses
  `ErrorCodes::ERR_INVALID_ARGUMENT`; `_BlendSpace.cpp` and `_AnimBlueprint.cpp` cite it zero times
  and carry many raw literals and are in no `PartiallyConvertedHandlerFiles` baseline, so they use
  the raw `TEXT("INVALID_ARGUMENT")` and stay non-adopting. `INVALID_ARGUMENT` was already
  registered (`ErrorCodes.h:538`); no new code. **Placement:** the composition is hoisted ABOVE the
  `LoadSkeletonFromPathAnim` call in the seven verbs that have one, which is what makes the
  regression test safe; each carries a comment saying not to move it back down. Regression coverage:
  `Tests/Gameplay/TestAnimationAuthoringNamePathSafety.cpp`, three new leaf ids
  `PinWright.animation.authoring.{create_animation_sequence,create_blend_space_1d,create_pose_library}.NameCarryingAPathIsRefused`
  — one verb per handler file, seven bad names each (`a//b`, rooted path, interior slash,
  backslash, `../Escape`, `..`, trailing slash) plus a bare-name control. `..` is there because
  `CreatePackage` has a SECOND Fatal — a name resolving to EMPTY through `ResolveName2`
  (`UObjectGlobals.cpp:1118`), not the `//` check at `:1094-1096` — which a slash-only filter would
  miss; the shared helper covers it because `INVALID_OBJECTNAME_CHARACTERS` contains `.` and `:`.
  Two `path`-side cases were added after a sibling agent's correction:
  `ExpectDoubleSlashFolderRefused` (`path: "/Game//Animations"` with a bare `name` —
  `NormalizeAnimPath` does not collapse an interior `//`, so the FOLDER was an equally live
  one-argument kill that the ticket's per-site notes do not spell out) and
  `ExpectTrailingSlashFolderStillAccepted`, which pins that a trailing-slash folder is still
  ACCEPTED. That second one addresses the sibling's other warning —
  `PinWrightComposeAssetPackagePath` composes with `Printf("%s/%s")` and would double a separator
  where `FString::operator/` does not, so routing a site through it blind can start refusing a
  folder that works today. **Not a hazard at these nine:** `Path` is
  `AnimationAuthoringHelpers::NormalizeAnimPath(...)` output at every one of them, and that
  function runs `while (EndsWith("/")) LeftChopInline(1)`, so a trailing slash provably cannot
  reach the helper. No trim was added; the test pins the property instead. The ticket's
  "no guard at all" verdict on these nine was re-verified rather than assumed: the three files
  contain no `SanitizeProjectRelativePath` / `IsValidAssetPath` / `ValidateAssetCreationPath` /
  `SanitizePackageName` / `IsValid*LongPackageName` call anywhere, before or after this change.
  `check_test_ids.py` re-run on the shared tree: CLEAN, 4855 ids.
  **Fatal-unreachability:** every bad name is paired with a well-formed `skeletonPath` naming no
  asset (fresh GUID under `/Game/PinWrightMissing/`). On the fixed build the compose check sits
  above the skeleton load and answers `INVALID_ARGUMENT`; on a reverted build the skeleton load is
  above the concatenation and answers `SKELETON_NOT_FOUND`, so the test goes red on the wrong code
  with the process alive and `CreatePackage` is unreachable on both. `create_ik_retargeter` and the
  `create_control_rig` fallback are deliberately NOT driven — they resolve nothing before their
  composition, so there is no earlier refusal to catch a reverted build and a bad name would reach
  the Fatal; they are fixed the same way and left uncovered rather than covered by a test that is
  only safe while the fix is present. Doc: one `##` section (above the first `###`) in
  `Docs/wiki-src/animation.authoring.md`. Noted, not touched: `FString FullPath = Path / Name;` in
  the `create_control_rig` fallback branch is pre-existing dead code, and that overlay page is
  ~25 KB, over the ~20 KB soft guideline (23.6 KB before this edit). Not compiled and not run: the
  orchestrator builds after the wave.
- `#4-assetutils-two-shared-helpers-guarded` `OPEN` developer — Closed the two
  `Utils/AssetUtils.cpp` sites only; status left `OPEN` because the rest of the sweep is in flight.
  **Sites re-derived at working-tree HEAD, both unchanged: `:1502`**
  (`PrepareBlueprintPackageGuardingNameCollision`, which passes ONLY its `PackagePath` to
  `CreatePackage` — its `AssetName` feeds the collision probe and is never concatenated) **and
  `:1878`** (`McpCreateControlRigBlueprint`, which composes `NormalizedPath / AssetName` and passes
  the result).
  **Complete caller enumeration** across all of `Source/`, gated sub-modules included
  (`PinWrightGeometry`, `PinWrightPCG`, `PinWrightChooser`, `PinWrightPoseSearch`,
  `PinWrightCommonUI`): `PrepareBlueprintPackageGuardingNameCollision` has exactly ONE production
  caller — `Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp:603`
  (`animation.authoring.create_anim_blueprint`), passing `Path / Name` composed from two raw
  `Ctx.GetString` values, so caller-supplied and unsafe; it is not among the nine sites `#3` closed.
  `McpCreateControlRigBlueprint` has ZERO production callers today (the `:3114` hit is a comment;
  that `#elif` branch inlines its own `CreatePackage`, closed by `#3`) plus ~40 test call sites in
  `Tests/Assets/TestCRIR*.cpp` and `CRIRTestHelpers.h`, all passing internally-composed literals; it
  is still `Utils/AssetUtils.h`-declared shared surface, so it is guarded on the same terms.
  **Guard placed INSIDE both helpers, not at the call sites.** Both are shared, so one guard covers
  every present and future caller including the unsafe one; and neither helper answers a request —
  both already return `bool`/`nullptr` plus an `OutError` string — so guarding inside changes no
  caller's error-code style (the anim-blueprint caller keeps its `ASSET_EXISTS`/`PACKAGE_ERROR`
  split, and `bOutNameCollision` stays `false` for a malformed argument so the two remain
  distinguishable). Guarding at the call site would have meant editing a file another agent of this
  wave owns, for one caller, and leaving the exported helper unguarded for the next one.
  **`PinWrightComposeAssetPackagePath` deliberately NOT reused, for a shape reason rather than the
  composition reason `#2` gave:** it COMPOSES `<folder>/<name>`, while both sites already hold a
  finished package path (`:1502` is handed one with the bare name as a separate argument; `:1878`
  normalizes its folder and composes itself), so the fit is half at best. Its two engine rules are
  applied directly by one file-local `AssetUtils_ValidatePackageTargetForCreatePackage`
  (distinctive name — Unity build) used by both sites: `FName::IsValidXName` +
  `INVALID_OBJECTNAME_CHARACTERS` on the bare name,
  `FPackageName::IsValidLongPackageName(..., bIncludeReadOnlyRoots=true, &Reason)` on the finished
  path, both engine reason texts verbatim, one `Warning` log (never `Error`) so a caller that drops
  `OutError` still leaves a record of which rule broke. Note `Utils/` → `Handlers/` includes already
  exist here (`AssetUtils.cpp:10`, plus 7 other `Utils/*.cpp`), so pulling the header down would not
  have been a NEW layering inversion — it was rejected on fit, not on direction.
  **Regression test** `Tests/Assets/TestAssetUtilsPackageTargetSafety.cpp`, two new leaf ids
  (`PinWright.Assets.AssetUtils.BlueprintPackageTargetRefusesMalformedPathOrName`,
  `...ControlRigCreateRefusesMalformedPathOrName`), driving both helpers as PURE FUNCTIONS — no
  verb, no dispatcher, so there is no handler ordering to preserve and no host fixture involved.
  **Fatal-unreachability on a REVERTED build** (where `CreatePackage` IS reached) rests on those
  being the only two Fatal branches — a name containing `//` (`:1094-1096`) and one resolving to
  empty (`:1118`): no fixture string that reaches `CreatePackage` contains `//`, none composes one
  (every folder literal is mounted with no trailing slash; no bad name carries a leading slash), and
  none is empty. The `//`-bearing NAME case IS driven at `:1502` — safely, because that helper never
  concatenates `AssetName` into the `CreatePackage` argument, verified at the call site — and it is
  the measured kill shape at the argument that carries it one level up. At `:1878`, which does
  concatenate, the bad names carry a single `/` or a `\` instead and a `//` case is deliberately
  absent. A reverted build therefore SUCCEEDS (a stray in-memory package, or a real CR BP the
  fixture tears down) and the assertions go red with the process alive. Stated rather than implied:
  the `//` input at `:1878` is covered by no test, only by the single `IsValidLongPackageName` call
  that also rejects the unmounted-root and `INVALID_LONGPACKAGE_CHARACTERS` cases that ARE asserted.
  Each test ends with a valid-input control. `check_test_ids.py` re-run on the shared tree: CLEAN,
  4832 ids, no dot-prefix collisions, no duplicates. No new error code — `AssetUtils.cpp` emits none
  at all, it returns strings — so `Handlers/ErrorCodes.h` is untouched. No doc change: neither
  helper is a verb, and the one namespace page affected (`animation.authoring`) was already updated
  by `#3`. Not compiled and not run per instruction; the orchestrator builds after the wave.
  **Follow-up for whoever owns `AnimationAuthoringHandler_AnimBlueprint.cpp`:** its `:603` caller
  discards `GuardError` and answers `PACKAGE_ERROR`/"Failed to create package" for what is now an
  argument refusal. The crash is closed either way, but the message is misleading and wants
  `INVALID_ARGUMENT` plus the engine reason. Left alone deliberately — that file is another agent's
  in this wave.
  **Two cross-agent corrections checked against this fix rather than assumed away.** (1) The SECOND
  Fatal (`:1118`, empty after `ResolveName2`) is excluded here by a separate argument, not by the
  `//` one: `ResolveName2` (`UObjectGlobals.cpp:1219-1240`) walks `.` and `:` delimiters and returns
  the name UNCHANGED the moment it finds neither, so a dotless, colonless name resolves to itself
  and cannot empty. **No fixture string reaching `CreatePackage` in the new test contains `.` or
  `:`** (the GUID suffixes are `EGuidFormats::Digits`, hex only); the test header now states that
  and forbids adding one, and `..` — the known killer of that branch — appears nowhere in it.
  (2) The trailing-slash OVER-refusal that `PinWrightComposeAssetPackagePath`'s `Printf("%s/%s")`
  can cause does not arise here, verified rather than reasoned around: that helper is not used,
  `:1878` strips every trailing slash from the folder (`AssetUtils.cpp:1945-1948`) BEFORE the new
  check runs, and `:1502` validates the finished path as given, which its one caller composes with
  `FString::operator/` — never a trailing slash from a non-empty name. The `PackagePath`-as-given
  contract is now stated in `AssetUtils.h` so a future caller does not hand it a folder. On treating
  the ticket's per-site classifications as claims rather than facts: both of this file's entries
  were re-read at the call site and both were accurate — `:1502` passes only `PackagePath` to
  `CreatePackage`, `:1878` passes the composition, and neither carried any guard.
- `#5-aihandler-four-sites-guarded` `OPEN` developer — Closed the four
  `Handlers/AI/AIHandler.cpp` sites. Line numbers re-derived on the shared tree (the ticket's were
  stale): `:248` → `CreateBlackboardAsset`, `:763` → `ai.create_state_tree`, `:1145` →
  `ai.create_smart_object_definition`, `:1640` → `ai.create_mass_entity_config`. **Both of the
  ticket's classifications re-verified in source rather than trusted**, and both hold: the
  blackboard site did sanitise the folder through `SanitizeAIAssetPath` and then concatenate the
  caller's `name` onto it raw, and the other three were `Ctx.GetString(path) / Ctx.GetString(name)`
  with nothing argument-driven above the concatenation at all.
  **Guard.** All four route through one new file-local `PinWrightAiComposeCreatePackagePath`
  (distinctive name, file-scope `static`, Unity-safe) that trims a trailing `/` off the folder —
  `FString::operator/` tolerated one and the shared composer's `Printf("%s/%s")` would turn it into
  the `//` it exists to reject — then delegates to `PinWrightComposeAssetPackagePath` and refuses
  with `INVALID_ARGUMENT` carrying the engine's reason verbatim. The shared helper was used, not a
  hand-rolled filter, precisely because `INVALID_OBJECTNAME_CHARACTERS` covers `.` and `:` as well
  as `/` — a `..` name reaches the SECOND Fatal (`:1118`, empty after `ResolveName2`), which a
  slash-only filter would miss. Three sites are the three-line drop-in. The blackboard site needed
  more: the compose was hoisted into the `ai.create_blackboard_asset` handler ABOVE
  `CreateBlackboardAsset`, whose signature now takes the validated `FullPath` instead of
  re-deriving one from the raw arguments — leaving the old `SanitizeAIAssetPath`+concat in place
  would have validated one string and passed a different one, i.e. this ticket's own
  "computed and then discarded" shape. The engine check that replaced that sanitiser is strictly
  stronger on every input it covered (traversal, `\`, `~`, unmounted root) and additionally refuses
  `//` rather than silently repairing it. `SanitizeAIAssetPath` stays live for `ai.create_blackboard`
  (`:2388`, already guarded — the whole composed path is sanitised there; untouched).
  **Error code:** `AIHandler.cpp` references `ErrorCodes::` zero times, so it has not adopted the
  registry and a raw `TEXT("INVALID_ARGUMENT")` is correct here (`RegistryAdoptingFilesUseConstantsOnly`
  is per-file); the code is already registered as `ERR_INVALID_ARGUMENT` in `Handlers/ErrorCodes.h`,
  so `AllEmittedCodesAreRegistered` is satisfied and that header is untouched.
  **Regression test** `Tests/Gameplay/TestAiCreateAssetNamePathSafety.cpp`, four new leaf ids
  (`PinWright.ai.{create_blackboard_asset,create_state_tree,create_smart_object_definition,create_mass_entity_config}.NamePathSafety`),
  driving the real registered handlers. **Fatal-unreachability, stated exactly because it differs
  per site.** No case anywhere sends a name containing `//`, and none needs to: `/` is in
  `INVALID_OBJECTNAME_CHARACTERS` (`Core/Public/UObject/NameTypes.h:191`), so `FName::IsValidXName`
  refuses `Sub/Leaf` and `a//b` by the same predicate for the same reason — the survivable member
  of the class proves the lethal one. Every refusal case pairs its bad name with an UNMOUNTED
  folder (`/PinWrightMissingRoot/AiNameSafety`). At `ai.create_blackboard_asset` that makes
  `CreatePackage` strictly UNREACHABLE on a reverted build: the pre-fix path ran
  `SanitizeAIAssetPath` on the folder first and its `IsValidMountPoint` refuses an unmounted root,
  so the call answers `CREATION_FAILED` ABOVE the concatenation and the `INVALID_ARGUMENT`
  assertion goes red with the process alive — which is why the fix's compose must stay above that
  helper, and the handler carries a comment saying so. At the other three there is nothing
  argument-driven above the concatenation, so a reverted build DOES reach `CreatePackage` —
  deliberately, with an argument that cannot enter either Fatal branch: every candidate is
  `<unmounted folder>/<bad name>`, none contains `//` (`PathAppend` does not double, and no name
  carries one), none is empty, and the four shared names are dot-free so `ResolveName2`'s splitting
  loop returns on its first iteration (`:1236-1240`) and cannot shorten one to empty. The single
  dotted case (`../Escape`) is driven ONLY at the blackboard verb, where `CreatePackage` is never
  called at all. The unmounted root also makes the follow-up save a no-op, so a reverted build
  leaves nothing on disk; it answers success or a downstream failure, never `INVALID_ARGUMENT`.
  A `//`-bearing name is therefore covered by no test at any of the four verbs — only by the
  `FName::IsValidXName` call that also rejects the slash cases that ARE asserted. Each verb ends
  with a valid-input control (bare name + real mounted folder) so a fix that refused everything
  fails; the control runs FIRST because it doubles as the host probe — `PLUGIN_DISABLED`
  (SmartObjects / MassGameplay) and the state-tree `headersUnavailable` fake-success are both
  skipped through `PinWrightTestSkip::SkipAssertions`, not passed. The reflection-only `ai.*`
  SmartObject/Mass paths were not touched.
  **Checks re-run on the shared tree:** `check_test_ids.py` CLEAN (4850 ids, no dot-prefix
  collisions, no duplicates); `check_test_skips.py` CLEAN. **Doc:** one new `##` section in
  `docs/wiki-src/ai.md` ("Asset names are bare names, not paths"), placed above the first `###`
  so it renders. Not compiled and not run per instruction; the orchestrator builds after the wave.
  **Observed while here, deliberately left alone:** `ai.create_state_tree`'s
  `#if MCP_STATE_TREE_HEADERS_AVAILABLE` `#else` branch fake-succeeds with `headersUnavailable:true`
  instead of erroring — an honesty defect unrelated to this ticket and out of its scope.
- `#6-audio-cluster-fifteen-sites-one-shared-guard` `OPEN` developer — Closed the AUDIO cluster:
  the thirteen `Handlers/Audio/AudioAuthoringHandler.cpp` sites and the two
  `Handlers/Audio/MetaSound/MetaSoundPatchPresetHandler.cpp` sites. Status left `OPEN` — the rest
  of the sweep is in flight. **Line numbers re-derived at working-tree HEAD and matched the ticket
  exactly** (`AudioAuthoringHandler.cpp` 354/443/857/919/954/1545/1771/2043/2270/2327/2523/2582/2778,
  `MetaSoundPatchPresetHandler.cpp` 78/218). The "no guard at all" verdict is **re-verified, not
  carried over**: all fifteen are the identical two-line idiom `FString PackagePath = Path / Name;`
  then `CreatePackage(*PackagePath)`, with `Name` straight from `Ctx.RequireString(TEXT("name"))` —
  which does an emptiness check and no character validation whatsoever
  (`HandlerContext.cpp:93-111`) — and `Path` from a folder-only normalizer. By verb:
  `create_sound_cue`, `create_sound_concurrency`, `create_metasound` (both the factory and the
  `#elif MCP_HAS_METASOUND` no-factory branch), `create_sound_class`, `create_sound_mix`,
  `create_attenuation_settings`, `create_dialogue_voice`, `create_dialogue_wave`,
  `create_reverb_effect`, `create_source_effect_chain`, `create_source_effect_preset`, the
  `CreateSoundSubmixAsset` static behind `create_sound_submix`, plus `create_metasound_patch` and
  `create_metasound_preset`.
  **ONE SHARED GUARD, and the judgement is affirmative because the rule is genuinely identical at
  all fifteen** — every site reads the same two arguments (`name`, `path`), composes them the same
  way, and hands `Name` to `FName(*Name)` as the UObject name, so a path-shaped name is wrong at
  every one of them for the same reason. New named-namespace header
  `Handlers/Audio/AudioPackagePathGuard.h`
  (`PinWrightAudioPackagePath::ComposeAudioAssetPackagePathOrRefuse`, grep-verified unique
  plugin-wide; named namespace plus `inline` per the audio cluster's existing Unity-ODR convention)
  wraps the plugin-wide `PinWrightComposeAssetPackagePath` and sends the `INVALID_ARGUMENT` refusal
  itself, so each site is two lines and there is one place to fix. The only per-site variation is
  the return statement (`return nullptr` in the submix static, `return true` in the twelve
  handlers), which is the caller's business, not the rule's.
  **The `Printf`-doubling trap `#2` flagged is handled INSIDE the wrapper, not assumed away.**
  `PinWrightComposeAssetPackagePath` composes with `Printf("%s/%s")` and so doubles a folder that
  already ends in `/`, which `IsValidLongPackageName` then rejects — whereas `FString::operator/`
  (`PathAppend`, `String.cpp.inl:855-885`, re-read: it POPS the terminator on that branch) does
  not. Routing thirteen verbs through an over-strict guard would break thirteen at once, so the
  wrapper trims trailing `/` before composing. Both current normalizers (`NormalizeAudioPath`,
  `MetaSound::NormalizeAudioAssetPath`) already strip trailing slashes in a loop, so the divergence
  is unreachable through a live verb today — the trim exists so this helper's contract stops
  depending on two other functions continuing to do that. A trailing BACKSLASH is deliberately not
  trimmed: `INVALID_LONGPACKAGE_CHARACTERS` refuses it either way, with a truer reason.
  **Placement:** replaced in place at fourteen sites (the composition was already the first thing
  after the parameter reads). Moved ABOVE the pre-existing `resolutionRule` resolution at
  `create_sound_concurrency` only, because that ordering is what makes the regression test safe;
  the handler carries a comment saying not to move it back. `create_source_effect_preset` and
  `create_metasound_preset` keep their existing `effectClass` / `referencedSource` checks above the
  guard — minimal diff, and the guard is still above every composition.
  **Error-code adoption checked per file, independently:** both files already cite `ErrorCodes::`
  AND carry many raw literals, and both are in `PartiallyConvertedHandlerFiles`
  (`TestErrorCodeRegistry.cpp:391-392`), so neither is held to the no-hand-spelling rule and
  neither is flipped by this change. The new refusals use `ErrorCodes::ERR_INVALID_ARGUMENT`
  regardless; it was already registered (`ErrorCodes.h:538`), so `Handlers/ErrorCodes.h` is
  untouched. The new header is NOT baselined and is clean: it cites the registry and spells no code
  by hand. No `UE_LOG` added — refusals travel through `Ctx.SendError` only, so nothing logs at
  `Error`.
  **Regression coverage:** `Tests/Media/TestAudioCreatePackagePathSafety.cpp`, three new leaf ids —
  `PinWright.audio.authoring.create_sound_concurrency.NameCarryingAPathIsRefused`,
  `PinWright.audio.authoring.package_path_guard.EveryCreatePackageIsGuarded`,
  `PinWright.audio.authoring.package_path_guard.TrailingSlashFolderComposesWithoutDoubling`.
  `check_test_ids.py` CLEAN (4855 ids on the shared tree, no dot-prefix collisions, no duplicates);
  `check_test_skips.py` CLEAN.
  **Fatal-unreachability, on both a fixed and a reverted build.** The driven verb is
  `create_sound_concurrency`, chosen because it is host-independent (core-engine
  `USoundConcurrency`, no feature `#if`, no plugin/asset/registry fixture) — so this file has NO
  conditional-skip path and emits no `PINWRIGHT_ASSERTIONS_SKIPPED`. Every bad `name` is paired
  with `resolutionRule: "PinWrightNotAResolutionRule"`, well-formed text naming no rule in the
  verb's closed vocabulary. On the FIXED build the guard sits above the rule check and answers
  `INVALID_ARGUMENT`. On a REVERTED build the pre-existing "resolve resolutionRule up front" check
  — which is itself above the concatenation — answers `INVALID_RESOLUTION_RULE`, so the `TestEqual`
  goes red on the wrong code with the process alive and `CreatePackage` is never reached on either
  build. Seven bad names: `Bus//Master` (the one-argument kill at `:1094-1096`), a rooted path, an
  interior slash, a backslash (caught by the composed-path half, not the name half), `../Escape`,
  `Trailing/`, and bare `".."` — the input that reaches the **second** Fatal at `:1118` via
  `ResolveName2`, caught here because `INVALID_OBJECTNAME_CHARACTERS` contains `.`. Plus a
  bare-name control that must still reach `INVALID_RESOLUTION_RULE`, proving the refusal is not
  blanket. The `FindObject` "creates no object" assertions cannot themselves fatal:
  `StaticFindObject` calls `ResolveName2` with `Create=false` and `CreatePackage` sits strictly
  inside the `Create==true` branch (`UObjectGlobals.cpp:1278-1310`), and `StaticFindFirstObject`
  logs only on ambiguity, not on a miss (`:811`) — both read in source, not assumed.
  **The other fourteen sites are covered structurally, not by driving them:** the
  `EveryCreatePackageIsGuarded` source scan asserts, per file, that the code-line count of
  `CreatePackage(` equals that of `ComposeAudioAssetPackagePathOrRefuse(` (13/13 and 2/2 measured),
  that the old `PackagePath = Path / Name` idiom is gone, and that the scan still reaches at least
  15 sites in total — so a sixteenth verb added with the old idiom goes red.
  `TrailingSlashFolderComposesWithoutDoubling` exercises the wrapper directly, because a guard
  stricter than the composition it replaced is invisible to every end-to-end test on this cluster
  while the normalizers keep stripping.
  **Not driven deliberately:** the two MetaSound verbs, whose earlier gates need a MetaSound
  registry this host reports as `metasound-registry-uninitialized`; they are fixed identically and
  covered by the scan, which reads files from disk and needs no registry — rather than by a live
  test that would skip.
  Doc: one `##` section (above the first `###`) in `Docs/wiki-src/audio.authoring.md`, stating the
  bare-name/folder split once for the whole namespace; a verb enumeration was trimmed back out
  because it duplicated the auto-generated `## Methods` index. That page is now 20.1 KB, marginally
  over the ~20 KB soft guideline — noted, not acted on, and the RENDERED namespace page is far
  smaller since `###` sections do not render there. Not compiled and not run per instruction; the
  orchestrator builds after the wave.
- `#7-gated-sub-module-three-sites-were-already-guarded` `OPEN` developer — Closed the three gated
  sub-module sites. **The headline is a correction: all three were listed under "no guard at all"
  and none of them was unguarded.** Both files route every `CreatePackage` through a file-local
  `BuildCreatePaths` whose last act before publishing the path is
  `FPackageName::IsValidLongPackageName(OutPackagePath, /*bIncludeReadOnlyRoots=*/false, &Reason)`,
  which `IsValidTextForLongPackageName` (`PackageName.cpp:1682-1714`, read in source) makes reject
  `//`, the too-short/empty name, a missing leading slash, a trailing slash and
  `INVALID_LONGPACKAGE_CHARACTERS`. Both Fatals were therefore already unreachable at all three
  sites. This is not a concurrent agent's fix arriving first: `git diff` on both files was empty at
  pick time, so the guard is present at the filing commit and the reporter's verdict is a mis-read —
  the third one this ticket has produced, after `#2`'s and `#4`'s. Sites re-derived (the ticket's
  numbers were stale and the directory stems elided):
  `PinWrightChooser/Private/Handlers/Chooser/ChooserAuthoringHandler.cpp:962` (`chooser.create`, now
  `:989`) and `PinWrightPoseSearch/Private/Handlers/PoseSearch/PoseSearchHandler.cpp:452` / `:575`
  (`pose_search.create_schema` / `create_database`, now `:459` / `:586`).
  **Shared header reachability, verified rather than assumed:** both `Build.cs` files carry
  `PrivateIncludePaths.Add(Path.Combine(ModuleDirectory, "..", "PinWright", "Private"))`, so
  `#include "Handlers/PackagePathCompose.h"` resolves from either sub-module exactly as the
  `Handlers/HandlerContext.h` include already sitting in both files does. Host gating verified too:
  this editor's startup line reads `loaded=[geometry,model,pcg,chooser,pose_search,ui] skipped=[]`
  (`Saved/Logs/PDS.log`, 2026-08-30 20:17), so both sub-modules' tests actually run here.
  **PoseSearch (2 sites): no functional change.** Both take ONE whole caller-supplied `assetPath`,
  which is precisely the shape the ticket says wants `IsValidLongPackageName` or
  `SanitizeProjectRelativePath` rather than the compose helper — and `BuildCreatePaths` already
  stacks three independent checks (`NormalizeAssetPath`, `SanitizeProjectRelativePath`,
  `IsValidLongPackageName`), all above the `CreatePackage` call. Adding a fourth would be dead code.
  Both sites got a comment naming the guard and forbidding the call being moved above it, so the
  next sweep does not re-flag them.
  **Chooser (1 site): one real hole, in the "folder guarded, caller's NAME unchecked" bucket, not
  the bucket it was filed in.** `BuildCreatePaths`' name branch sanitizes the FOLDER
  (`SanitizeProjectRelativePath`) and concatenated the caller's `name` onto it raw, so
  `name: "Sub/Leaf"` or `"/Game/X"` composed a package the caller never named and wrote the chooser
  there silently. Not fatal — the downstream `IsValidLongPackageName` caught `//` — but the same
  argument confusion one step short of it. Routed through the shared
  `PinWrightComposeAssetPackagePath`, refusing `INVALID_ARGUMENT` with the engine reason verbatim.
  **The `Printf("%s/%s")` doubling hazard `#2` and `#4` both flagged does not apply here, and the
  reason is structural rather than lucky:** `NormalizePackagePath` runs `SanitizeProjectRelativePath`
  (collapses `//` in a loop) and a pre-existing `while (Folder.EndsWith("/")) LeftChopInline(1)`
  loop sits directly above the call, so the folder can carry neither an interior `//` nor a trailing
  slash. Both are load-bearing, both are now commented as such, and a dedicated control case
  (`TrailingSlashFolder`) asserts a folder spelled `"/Game/X/"` still composes and is NOT refused —
  the regression `#2` correctly predicted a blind reuse would cause. The refusal is also not a dead
  end: `chooser.create`'s `path` is REQUIRED and is the whole package path when `name` is omitted,
  so a nested destination is still nameable.
  **Error-code adoption checked per file, independently:** `grep -c "ErrorCodes::"` is **0** in both
  sub-modules entirely, so both are raw-literal-only and stay that way — raw
  `TEXT("INVALID_ARGUMENT")`, no `ErrorCodes::` reference introduced, no file flipped into the
  registry test's scope. `INVALID_ARGUMENT` was already registered (`ErrorCodes.h:538`);
  `Handlers/ErrorCodes.h` is untouched and needed no new code.
  **Regression coverage, one file per sub-module so each compiles into the right DLL:**
  `PinWrightChooser/Private/Tests/Assets/TestChooserCreateNamePathSafety.cpp` (id
  `PinWright.chooser.CreateNameCarryingAPathIsRefused`) and
  `PinWrightPoseSearch/Private/Tests/Gameplay/TestPoseSearchCreateAssetPathSafety.cpp` (ids
  `PinWright.pose_search.CreateSchemaMalformedAssetPathIsRefused`,
  `...CreateDatabaseMalformedAssetPathIsRefused`). Flat leaf names matching each sub-module's
  existing convention. `check_test_ids.py` re-run on the shared tree after the edits: CLEAN, 4855
  ids, no dot-prefix collisions, no duplicates; `check_test_skips.py` CLEAN.
  **Fatal-unreachability.** Chooser: on the FIXED build nothing is ever composed —
  `INVALID_OBJECTNAME_CHARACTERS` (`NameTypes.h:191`) contains `/`, so `IsValidXName` refuses every
  path-shaped name before the join, and the join cannot double because the folder is chopped. On a
  REVERTED build the old `Folder / Name` runs, `PathAppend` does not double a leading slash, and the
  untouched `IsValidLongPackageName` below still refuses `a//b`, `..`, `Trailing/` and the backslash
  case as `INVALID_PATH`. For the residual cases a reverted build composes into a VALID path
  (`Sub/Leaf`, `/Game/X/Y`), every refusal case pairs its bad name with
  `contextObjectType: "/Script/Engine.DoesNotExist"` — a well-formed path in an ALREADY-LOADED
  script package naming no class — which `HandleCreate` resolves ABOVE `CreatePackage`, so the
  reverted build answers `CLASS_NOT_FOUND` and the test goes red on the wrong code with the process
  alive. PoseSearch: the three stacked checks mean removing any one leaves two, and every bad
  `assetPath` carries `//`, `..`, a drive letter or a bare mount root, which no single check owns
  alone; on top of that every case pairs its bad path with a well-formed companion (`skeleton` /
  `schema`) naming no asset (fresh GUID under `/Game/PinWrightMissing/`), resolved above
  `CreatePackage`, so even a fully gutted `BuildCreatePaths` answers `SKELETON_NOT_FOUND` /
  `SCHEMA_NOT_FOUND`. **Both Fatals are covered, not just the famous one:** `a//b` and the `//`
  paths aim at `:1094-1096`, while a bare `".."` name and a bare `/Game` mount root aim at `:1118`
  (empty after `ResolveName2`) — `INVALID_OBJECTNAME_CHARACTERS` covers `.` and `:` as well as `/`,
  and `INVALID_LONGPACKAGE_CHARACTERS` covers `.` too, so one guard turns away both classes. Every
  bad path is also given a GUID leaf so `NormalizeAssetPath`'s "retry the leaf under
  /Game|/Engine|/Script if that package EXISTS" rescue can never fire and make a refusal
  host-dependent. Each test ends with a valid-input control asserting the refusal is not blanket.
  **Finding the rest of the sweep should have: `CreatePackage` is not the only door.**
  `StaticLoadObjectInternal` calls `ResolveName2(..., Create=true, ...)` (`UObjectGlobals.cpp:1427`),
  which itself calls `CreatePackage(*PartialName)` on the raw package name at `:1310` when the
  package is not already loaded. So **every `LoadObject` on an unvalidated composed object path is
  the same Fatal**, and in both files here it sits on the `ALREADY_EXISTS` check one line ABOVE the
  `CreatePackage` the ticket enumerated — i.e. a guard "moved down to just before `CreatePackage`"
  would not actually be a guard. `FindObject` is safe: `StaticFindObject` resolves with
  `Create=false` (`:620`), which is why the tests' existence assertions can be driven with a
  `//`-bearing path. The ticket enumerates direct `CreatePackage` calls only; whoever scopes the
  follow-up should decide whether `LoadObject`-on-caller-text is a second sweep.
  **Doc:** one paragraph in `Docs/wiki-src/chooser.md` under the existing `### chooser.create` H3
  (a bare dotted method name), stating that `name` is a bare leaf and that a nested destination goes
  in `path`. No pose_search doc change — nothing about that verb's behaviour changed. Not compiled
  and not run per instruction; the orchestrator builds after the wave.
  **Amended for the folder dimension.** Checked rather than assumed, since a sibling found a
  normalizer that strips trailing slashes and maps `/Content` to `/Game` but does NOT collapse an
  interior `//`: **neither of my files has that shape.** Both route every folder and every whole
  path through `SanitizeProjectRelativePath`, whose collapse loop (`PathUtils.cpp:57-60`) rewrites
  `//` to `/` before anything is composed — chooser via `NormalizePackagePath` on BOTH branches
  (folder+name at `:60`, whole `path` at `:100`), pose_search via `NormalizePackagePath` after
  `NormalizeAssetPath`, which refuses a `//` path outright rather than collapsing it. So a
  `path: /Game//Chooser` with an impeccable bare name is collapsed to `/Game/Chooser`, not
  composed into a Fatal. The guards satisfy the practical rule independently of that: both validate
  the COMPOSED string with `FPackageName::IsValidLongPackageName` — inside
  `PinWrightComposeAssetPackagePath` on the chooser name branch, and at the end of
  `BuildCreatePaths` on every branch of both files — so if the sanitizer were ever swapped for a
  non-collapsing one the answer flips to `INVALID_ARGUMENT`/`INVALID_PATH`, never to a dead editor.
  Folder cases added to both sub-modules' own `Private/Tests/` trees, no new test ids
  (`check_test_ids.py` CLEAN, 4855; `check_test_skips.py` CLEAN): chooser gets a `//`-folder case
  on the folder+name branch AND a `//` case on the `path`-alone branch, both with a bare legal
  leaf, both asserting `CLASS_NOT_FOUND` — the collapse-and-proceed contract — so the file goes RED
  rather than fatal the day that sanitizer changes; pose_search gets a root-spelled `//Game/...`
  variant on both verbs alongside the existing mid-path `/Game/PinWrightTests//<leaf>` cases, which
  already carried the `//` in the FOLDER portion with a bare GUID leaf. Safe to drive on both a
  fixed and a reverted build: every one of these composes or refuses above `CreatePackage`, and the
  chooser cases reach only `ResolveUClass` on an already-loaded script package.
- `#8-skeleton-physics-three-sites-guarded` `OPEN` developer — Closed the three skeleton/physics
  sites. Status left `OPEN`; the rest of the sweep is in flight.
  **Files:** `Handlers/Animation/SkeletonHandler.cpp`, `Handlers/Physics/PhysicsHandler.cpp`,
  `Handlers/Animation/PhysicsAssetHandler.cpp`, plus new
  `Tests/Gameplay/TestSkeletonPhysicsPackagePathSafety.cpp` and `Docs/wiki-src/{physics,skeleton}.md`.
  **Sites re-derived at working-tree HEAD; all three line numbers were still exact**
  (`SkeletonHandler.cpp:654` `skeleton.create_skeleton`, `PhysicsHandler.cpp:212`
  `physics.setup_physics_simulation`, `PhysicsAssetHandler.cpp:331`
  `skeleton.create_physics_asset`) — but **one of the two classifications was wrong and is
  corrected here.**
  **`PhysicsHandler.cpp:212` — classification CONFIRMED, and it is the most reachable of the
  three.** `SavePath` went through `IsValidLongPackageName` + a `TryConvertFilenameToLongPackageName`
  fallback; `physicsAssetName` had no validation of any kind; the two met at
  `FString::Printf(TEXT("%s/%s"), *SavePath, *PhysicsAssetName)`. Because Printf doubles the
  separator unconditionally, this site was reachable by a **rooted** name (`"/Game/X"` →
  `/Game/Physics//Game/X`) as well as by the `//`-bearing name every unguarded site shares, and by
  `".."`, which reaches the *second* Fatal (`UObjectGlobals.cpp:1118`, empty after `ResolveName2`)
  rather than the double-slash one. Guard: `PinWrightComposeAssetPackagePath`, which also replaces
  the Printf composition. `SavePath` is trimmed of a trailing slash immediately before the helper —
  the helper itself joins with Printf, so an untrimmed folder would compose the very `//` it exists
  to refuse; the trim is placed AFTER the pre-existing savePath validation so the accepted set is
  unchanged.
  **`SkeletonHandler.cpp:654` — classification WRONG. It is not "folder guarded, name unchecked",
  and no input reaches the Fatal today.** The verb takes no name argument: `path`/`skeletonPath` is
  ONE whole caller path, already rejected pre-ticket for `..`, `//`, `\`, a non-mount root and
  anything failing `IsValidLongPackageName(path, /*readOnlyRoots=*/false)`. Because a surviving path
  can contain no `//`, no trailing slash and no `.` (which is in `INVALID_LONGPACKAGE_CHARACTERS`),
  the `GetPath() / GetBaseFilename()` recomposition handed to `CreatePackage` is provably the
  identity. The real defect is narrower: the validated string and the passed string are different
  strings, and the identity is a property of the three lines above the call rather than of the call.
  Guard: `FPackageName::IsValidLongPackageName(FullPackagePath, bIncludeReadOnlyRoots=true)` on the
  composed string itself, which by construction can refuse nothing the stricter check above already
  accepted — zero behaviour change, structural guarantee only.
  **`PhysicsAssetHandler.cpp:331` — classification CONFIRMED ("no guard at all").** `outputPath` was
  read raw and split/rejoined into `CreatePackage`. One-argument kills: `"//Game/X"` survives the
  split-and-rejoin as `//Game/X`; `"/"` splits into two empty halves and composes to `""`; `".."`
  composes to `"."`. Guard: `IsValidLongPackageName` on the composed string, **hoisted above the
  source-asset resolution** for the caller-supplied path, with the same check on the derived
  `<SourcePath>_PhysicsAsset` default at the point it is derived (deliberately not hoisted — a
  malformed `skeletalMeshPath` is better reported as the source-asset error).
  **Why not the shared helper on the two `operator/` sites:** `PinWrightComposeAssetPackagePath`
  joins with Printf, so routing an `operator/` site through it would newly *over-refuse* a
  trailing-slash folder that composes correctly today (`PathAppend`, `String.cpp.inl:855-885`, pops
  the separator instead of doubling). Both are whole-path sites, which the ticket already directs at
  `IsValidLongPackageName` directly.
  **Error-code adoption checked per file: all three cite `ErrorCodes::` ZERO times** and carry many
  raw literals, so all three use raw `TEXT("INVALID_ARGUMENT")` and stay non-adopting. No new code;
  `INVALID_ARGUMENT` was already registered (`ErrorCodes.h:538`). Refusals send only — no `UE_LOG`,
  matching the shipped Foliage/Landscape sites.
  **Regression coverage:** three new leaf ids —
  `PinWright.physics.setup_physics_simulation.NameCarryingAPathIsRefused` (7 bad names: rooted path,
  `Sub//Leaf`, interior slash, backslash, `../Escape`, `..`, trailing slash),
  `PinWright.skeleton.create_physics_asset.OutputPathIsValidated` (7 bad paths incl. `//Game/…`,
  `/`, `..`, `/Game/..`, rootless, unmounted root), and
  `PinWright.skeleton.create_skeleton.HazardousPathsAreRefused`. `check_test_ids.py` CLEAN (4855
  ids, no dot-prefix collisions, no duplicates); `check_test_skips.py` CLEAN.
  **Fatal-unreachability, on both a fixed and a reverted build:** every bad value is paired with a
  well-formed long package name naming NO asset (fresh GUID under `/Game/PinWrightMissing/`) in the
  argument each verb resolves FIRST — `meshPath` for the physics verb, `skeletalMeshPath` for
  `create_physics_asset`. Both fixes are hoisted ABOVE that resolution, so a fixed build answers
  `INVALID_ARGUMENT` while a reverted build answers `ASSET_NOT_FOUND` / `MESH_NOT_FOUND` from the
  load — which sits above the concatenation — and the `TestEqual` on `INVALID_ARGUMENT` goes red
  with the process alive. Both handlers carry a comment saying the placement is load-bearing. A
  bare-name / well-formed-path control asserts the refusal is not blanket by requiring the
  source-asset error, so it too reaches no creation code. `skeleton.create_skeleton` is the
  exception and its test says so in a header note: no input discriminates a fixed from a reverted
  build there, because the pre-existing checks refuse every hazardous path on both; the test pins
  those refusals so a future weakening of any of them turns red instead of turning an editor off.
  **Modal-hang argument (`B-physics-asset-factory-modal-hang`):** no case in the new file reaches
  physics-asset creation at all — every case is a refusal or a source-asset miss, so neither
  `UPhysicsAssetFactory` nor the headless replacement is entered, and no `PINWRIGHT_ASSERTIONS_SKIPPED`
  is needed (there is no fixture dependency). No valid-input control that creates a physics asset was
  added: `Tests/Gameplay/TestPhysicsAssetFactoryModalHang.cpp` already owns that coverage for both
  verbs through the factory-free path. `skeleton.create_skeleton`'s positive path is likewise already
  driven by `TestAnimationHandlers.cpp` as a fixture, so no package-writing control was duplicated here.
  **Doc:** `### physics.setup_physics_simulation` added to `Docs/wiki-src/physics.md` (which had no
  `###` before, so no `##` is swallowed) and `### skeleton.create_skeleton` +
  `### skeleton.create_physics_asset` appended to `Docs/wiki-src/skeleton.md`, all below the
  existing `##` sections.
  **Noted, not touched (another agent's file):** `Handlers/Environment/LandscapeHandler.cpp` cites
  `ErrorCodes::` 32 times yet emits a raw `TEXT("INVALID_ARGUMENT")` at the `meshPath` check two
  lines below its compose refusal — that shape is what
  `PinWright.core.error_codes.RegistryAdoptingFilesUseConstantsOnly` fails on. Not reverted, not
  edited; flagged for whoever owns that file.
  **AMENDED — the FOLDER is a live one-argument kill here, and this site's "folder guarded"
  classification is misleading.** `savePath`'s guard is `IsValidLongPackageName` (a validator,
  which does reject `//`) **but on failure it falls back to
  `TryConvertFilenameToLongPackageName` and assigns the result with no re-validation.** That
  fallback is a normalizer, not a validator: `FPaths::NormalizeFilename` is called with
  `bRemoveDuplicateSlashes=false` (`Paths.cpp:1332-1339`) and `InternalFilenameToLongPackageName`
  ends in `OutPackageName << Result`, returning the last attempted span — so
  `savePath: "/Game//Physics"` comes back out **verbatim and reporting success**. Read in
  `C:/UE_5.8/Engine/Source/`, not assumed. Pre-fix that composed a fatal package path with a
  perfectly BARE `physicsAssetName` — and with NO `physicsAssetName` at all, since the derived
  `<MeshName>_Physics` default takes the same route. It is stopped only because
  `PinWrightComposeAssetPackagePath` validates the COMPOSED path; a name-only character check
  would not have caught it. The other two sites take one whole path and validate the composed
  string directly, so the same property holds there. **None of the three routes a folder through
  `SanitizeProjectRelativePath`** (which does collapse `//` in a loop). Coverage added: two folder
  cases on the physics verb (`/Game//Physics` and `//Game/Physics`, each paired with a BARE
  well-formed name, asserting the refusal quotes the FOLDER) and one on `create_physics_asset`
  (`/Game//Physics/PA_Safety` — `//` entirely in the directory half, bare leaf).
  **AMENDED — `CreatePackage` is not the first door at the physics site, and the hoist turned out
  to be load-bearing for that too.** `LoadObject` -> `StaticLoadObjectInternal` calls
  `ResolveName2(..., Create=true)` (`UObjectGlobals.cpp:1427`), and `ResolveName2` calls
  `CreatePackage(*PartialName)` (`:1310`). `physics.setup_physics_simulation` has exactly the
  shape that makes that fatal: its already-exists branch runs
  `UEditorAssetLibrary::DoesAssetExist(PhysicsAssetObjectPath)` and then
  `LoadObject<UPhysicsAsset>(nullptr, *PhysicsAssetObjectPath)` on the COMPOSED path, both several
  lines ABOVE the enumerated `CreatePackage`. **A guard placed immediately before `CreatePackage`
  would have been no guard at all.** Both composition guards were already hoisted to the top of
  the handler for the regression test's sake, so they sit above it; re-verified by line number
  (`:102` and `:228` vs `:239`/`:248`/`:261`) and now commented at the existence check so they are
  not moved back down. The other two sites load nothing on the composed path. `FindObject` is the
  safe counterpart (`Create=false`, `:620`) and is what the no-leaked-object assertions use.
  **NEW DOOR FOUND, NOT FIXED — recommend a separate ticket; it is not in the 61.** The same
  `ResolveName2(Create=true)` fact makes every `LoadObject`/`StaticLoadObject` on **unvalidated
  caller text** an editor-kill, and this sweep is direct-`CreatePackage`-only by construction, so
  none of those are enumerated anywhere. In my files, ungated: `PhysicsAssetHandler.cpp`'s
  `LoadSkeletalMeshFromPathPhys(SkeletalMeshPath, ...)` and
  `AnimationAuthoringHelpers::LoadSkeletonFromPathAnim(SkeletalMeshPath)`, plus
  `PhysicsHandler.cpp`'s second `LoadObject<USkeleton>(nullptr, *SkeletonPath)` and
  `physics.setup_ragdoll`'s. Incidentally protected: `PhysicsHandler`'s first `meshPath` /
  `skeletonPath` loads, each behind a `DoesAssetExist` registry lookup, which creates nothing and
  cannot match a `//` path. Deliberately NOT fixed here: the correct guard differs per site
  (`LoadSkeletonFromPathAnim` intentionally accepts `/Content`-prefixed and backslash paths that
  `IsValidLongPackageName` would refuse, so validating the raw string over-refuses), the class
  spans the whole plugin, and patching two files would be inconsistent without being safe. My
  tests are unaffected: every source path they pass is a well-formed GUID path.
  Doc: the `savePath` normalizer fact added to `Docs/wiki-src/physics.md`.
  Not compiled and not run per instruction; the orchestrator builds after the wave.
- `#9-ui-widget-cluster-three-sites-guarded` `OPEN` developer — Closed the UI/WIDGET cluster: three
  sites in three files. Status left `OPEN`; the rest of the sweep is in flight.
  **Files:** `Handlers/UI/WidgetCreateHandler.cpp`, `Handlers/Editor/UtilityWidgetHandler.cpp`,
  `Handlers/UI/WidgetAuthoringUtils.cpp` (+ its `.h` for the changed nullptr contract), plus new
  `Tests/UI/TestWidgetCreatePackagePathSafety.cpp` and `Docs/wiki-src/{widget,editor}.md`.
  **Sites re-derived at working-tree HEAD; all three line numbers were still exact**
  (`WidgetCreateHandler.cpp:64` `widget.create_widget_blueprint`, `UtilityWidgetHandler.cpp:70`
  `editor.create_utility_widget`, `WidgetAuthoringUtils.cpp:71`
  `WidgetAuthoringHelpers::CreateAssetPackage`).
  **Both "folder guarded, name unchecked" classifications CONFIRMED by reading the code, not
  assumed.** Each verb runs `folder` through `SanitizeProjectRelativePath` AND assigns the result
  back over the variable (`Folder = SanitizedFolder;`), then composes `Folder / Name` with `Name`
  straight off the payload. Because `PathAppend` does not double, a *rooted* name is not a kill
  here (unlike the Printf sites) — but `name: "a//b"` is, at both, and so is `name: ".."`, which
  reaches the SECOND Fatal (`UObjectGlobals.cpp:1118`, empty after `ResolveName2`, whose delimiters
  are `.` and `:` — never `/`).
  **THE SHARED HELPER'S CALLER LIST IS EMPTY, which is the one substantive finding here.**
  `WidgetAuthoringHelpers::CreateAssetPackage` is declared in `Handlers/UI/WidgetAuthoringUtils.h`
  and defined in the `.cpp`, and `grep -rn CreateAssetPackage Source/` returns exactly five hits:
  the declaration, the definition, and **a separate file-local `static CreateAssetPackage` in
  `Handlers/Blueprint/BlueprintTypeDefinitionHandler.cpp:495` plus its two call sites (`:508`,
  `:566`)** — a different function, and the `:497` entry the ticket lists under another agent's
  file. `git log -S CreateAssetPackage -- Source/` shows only the squash, so it has been callerless
  since. **No RPC verb reaches that site today**; it is guarded anyway (the guard is what makes it
  safe for its first caller) and NOT deleted — removing pre-existing dead code is out of scope, but
  it is flagged here as a deletion candidate for whoever owns that file next.
  **Guards.** The two verbs: `PinWrightComposeAssetPackagePath` on `(folder, name)`, replacing the
  `operator/` composition. The folder's trailing slash is popped immediately before the call
  (`RemoveFromEnd(TEXT("/"))`) because the helper joins with `Printf("%s/%s")` — the sanitizer
  preserves a trailing slash, so an untrimmed folder would compose the very `//` the helper exists
  to refuse, which would newly over-refuse `folder: "/Game/UI/"` (it works today via `PathAppend`).
  The mount-point fallback was moved from the composed path to the folder, which is
  behaviour-identical in every reachable case: after the sanitizer, `Folder` is either empty or
  already a mounted path, and empty still yields `/Game`. The shared helper: one whole
  caller-supplied path, so `FPackageName::IsValidLongPackageName(..., bIncludeReadOnlyRoots=true)`
  directly, applied AFTER the existing `/Game/` prepend — deliberately, because that prepend is
  itself a manufacturer of the Fatal (an unmounted `"/Foo/Bar"` becomes `"/Game//Foo/Bar"`, i.e.
  the function creates `//` out of an input that carried none). It returns `nullptr` with one
  `UE_LOG(..., Warning, ...)` naming the engine's reason; nothing logs at Error.
  **One extra ordering change, made only because the test could not be built safely without it.**
  In `WidgetCreateHandler.cpp` the bare-name half is hoisted ABOVE the folder sanitizer as a direct
  `FName::IsValidXName(Name, INVALID_OBJECTNAME_CHARACTERS, &Reason)`; the composed-path half stays
  at the composition. That verb has no other above-`CreatePackage` bail except the folder block, so
  without the hoist there is no payload that makes a reverted build stop before the concatenation.
  `editor.create_utility_widget` needed no hoist: its parent-class resolution already sits between
  the folder check and `CreatePackage`. Both handlers carry a comment saying the placement is
  load-bearing and naming the test.
  **Error-code adoption checked per file: all three cite `ErrorCodes::` ZERO times**, so both verbs
  use raw `TEXT("INVALID_ARGUMENT")` and stay non-adopting; the shared helper sends nothing at all
  (bool/nullptr + log), which sidesteps the trap entirely. No new code — `INVALID_ARGUMENT` is
  already registered (`ErrorCodes.h:538`).
  **Regression coverage:** three new leaf ids in one file —
  `PinWright.widget.create_widget_blueprint.NameCarryingAPathIsRefused`,
  `PinWright.editor.create_utility_widget.NameCarryingAPathIsRefused`,
  `PinWright.widget.authoring_utils.CreateAssetPackageRefusesInvalidPaths`. Helpers live in a named
  namespace (`WidgetCreatePackagePathSafetyHelpers`) per the cluster's Unity/ODR convention.
  `check_test_ids.py` CLEAN (4855 ids, no dot-prefix collisions, no duplicates);
  `check_test_skips.py` CLEAN.
  **Fatal-unreachability, on both a fixed and a reverted build — three different constructions,
  one per site.** (1) `widget.create_widget_blueprint`: every bad `name` is paired with
  `folder: "/Game/../../Engine/Content"`, which `SanitizeProjectRelativePath` rejects, so a
  reverted build answers `SECURITY_VIOLATION` from the folder block — above the concatenation — and
  the `TestEqual` on `INVALID_ARGUMENT` goes red with the process alive. The one case that cannot
  use that lever is the backslash (it survives `INVALID_OBJECTNAME_CHARACTERS` and is caught by the
  package rule *below* the folder block), so it is driven with a valid folder instead, and that is
  safe on its own terms: a backslash composes neither `//` nor an empty name, so a reverted build
  merely creates an asset and fails the `TestFalse`. (2) `editor.create_utility_widget`: every bad
  `name` is paired with a well-formed `parentClass` naming no existing class (fresh GUID under
  `/Game/PinWrightMissing/`), so a reverted build is refused `INVALID_PARENT_CLASS` at the
  parent-class resolution, above `CreatePackage`. (3) The shared helper has no second argument and
  therefore no lever, so its negative cases are restricted to inputs invalid for
  `IsValidLongPackageName` yet harmless for `CreatePackage` — a backslash and a trailing slash,
  neither of which composes `//` or resolves to empty (`ResolveName2` splits on `.`/`:`, never `/`,
  so a trailing slash survives non-empty). The file header forbids adding a `//`, `..` or
  unmounted-root case there in as many words, and says why each would end a live editor.
  **Controls:** a bare-name valid-input control per verb (creates the asset under
  `/Game/PinWrightTests/WidgetCreatePathSafety`, asserted via `DoesAssetExist`, torn down with
  `CleanupTestAsset` in `ON_SCOPE_EXIT`), plus a valid-path control on the helper, plus an ORDERING
  WITNESS asserting a good name with the rejected folder still returns `SECURITY_VIOLATION` — so
  the hoist added a check rather than replacing one.
  **Doc:** `### widget.create_widget_blueprint` added to `Docs/wiki-src/widget.md` immediately
  before the first pre-existing `###` (so no `##` section is newly swallowed), and
  `### editor.create_utility_widget` to `Docs/wiki-src/editor.md` likewise. Cross-references
  between the two are plain prose inside those `###` bodies, never a foreign-namespace `###`.
  **Amended after the wave's folder-dimension and LoadObject-door corrections, both re-checked
  against source rather than accepted.** (1) **`folder` is NOT a live kill at these two sites, and
  it is now pinned that way.** `SanitizeProjectRelativePath` is a validator, not merely a
  normalizer: it collapses `//` in a `while` loop (`Utils/PathUtils.cpp:56-60`), rejects `..`,
  forces a leading slash and requires a mounted root, so `folder: "/Game//Widgets"` reaches the
  composition already collapsed. Independently, both sites route the COMPOSED string through
  `PinWrightComposeAssetPackagePath`, i.e. `IsValidLongPackageName` on the final path, so a
  name-only check is not what is relied on. Two acceptance pins were added - an interior-`//`
  folder and a trailing-slash folder must both still SUCCEED - because the Printf join could
  otherwise have turned the crash fix into an over-refusal. (2) **Guard placement re-audited
  against `StaticLoadObjectInternal` -> `ResolveName2(Create=true)` (`:1427`) -> `CreatePackage`
  (`:1310`).** None of the three sites loads the composed path before creating it - there is no
  already-exists probe on any of them - and every guard sits above every intervening call, so no
  guard is defeated by that door. **Three instances of that door were nonetheless found in this
  cluster and are deliberately NOT patched here, because a one-file patch on a class this broad
  advertises coverage that would not exist:** `WidgetAuthoringHelpers::LoadWidgetBlueprint`
  (`WidgetAuthoringUtils.cpp:197,203`) hands a caller `widgetPath`, normalized only by a `/Game/`
  prepend, straight to `StaticLoadObject` - high reach, it is the widget cluster's loader;
  `UtilityWidgetHandler.cpp:86` hands the caller's `parentClass` to `LoadClass`; and
  `ResolveUClass` (`Utils/ClassUtils.cpp:113`) does `LoadObject<UClass>` on raw caller text and is
  shared by dozens of verbs, including the other file in this cluster. Each is a one-argument kill
  by `"//"`. They want their own ticket and their own sweep.
  Not compiled and not run per instruction; the orchestrator builds after the wave.
- `#10-gas-gameframework-blueprinttypes-three-sites` `OPEN` developer — Closed the three sites in
  `Handlers/Systems/GASHandler.cpp`, `Handlers/Systems/GameFrameworkHandler.cpp` and
  `Handlers/Blueprint/BlueprintTypeDefinitionHandler.cpp`. Status left `OPEN`; the rest of the
  sweep is in flight. Numbered `#10` because earlier numbers were claimed concurrently by the time this
  one landed. **Line numbers re-derived on the shared tree; all three still matched the ticket
  (`:2458`, `:98`, `:497`), but two of the three CLASSIFICATIONS did not.**
  **`GASHandler.cpp:2458` — `gas.create_ability_set`; the ticket's verdict was right and this was
  the only genuinely reachable-lethal one of my three.** `setPath`/`assetPath` is one whole caller
  path, so it is guarded with `FPackageName::IsValidLongPackageName(..., bIncludeReadOnlyRoots=true,
  &Reason)` directly rather than with the folder+name composer, refusing `INVALID_ARGUMENT` with
  the engine reason verbatim. **Placed after the `IsValidMountPoint` fallback and ABOVE the existing
  `LoadObject` existence check, not merely above `CreatePackage`; both halves of that are
  load-bearing.** After the fallback, because the fallback is itself a second kill shape:
  `IsValidMountPoint` refuses any rooted path outside `/Game|/Engine|/Script` that is not a mounted
  long package name, and the next line prepends `/Game/` onto it, so `"/NotAMount/X"` became
  `"/Game//NotAMount/X"` from one argument. Above the `LoadObject`, because that call is a SECOND
  door to the same Fatal — read in engine source, not assumed: for a path with no `.`,
  `StaticLoadObjectInternal` retries as `"<path>.<shortname>"` and `ResolveName2` then calls
  `CreatePackage` on the package half itself (`UObjectGlobals.cpp:1297-1311`), so a guard sitting
  immediately above `CreatePackage` would have left the verb exactly as lethal while reading as
  fixed. `GASHandler.cpp:772` (`CreateGASBlueprint`) was checked while here and is genuinely
  guarded (`ValidateAssetCreationPath` + `IsValidAssetPath`), consistent with its absence from the 61.
  **`GameFrameworkHandler.cpp:98` — classified "no guard at all"; true, but the site is DEAD CODE
  and no verb reaches it.** `CreateGameFrameworkBlueprint` has exactly one caller,
  `GF_CREATE_CLASS_HANDLER`, and that macro is DEFINED AND NEVER INSTANTIATED anywhere in
  `Source/`: the file registers nine `game_framework.configure_*` / `set_respawn_rules` /
  `get_game_framework_info` verbs and no `create_*` verb at all, so `CreateGameFrameworkBlueprint`,
  `FCommonParams::ExtractSavePath` and `FCommonParams::Path` are all unreachable from the wire.
  **The assignment's `ExtractSavePath` hint does not apply:** it never sees the bare `name`, and the
  folder it sanitises is re-normalised inside the create helper afterwards, so the composition point
  is the only correct home. Guarded there with the shared `PinWrightComposeAssetPackagePath` on
  `(FullPath, Name)`; the refusal travels out through the existing `OutError` and would read
  `CREATION_FAILED` at the hypothetical caller rather than `INVALID_ARGUMENT` — accepted
  deliberately rather than reordering a macro nobody expands.
  On the sibling agent's trailing-slash correction: it does not bite here, and this was verified
  rather than reasoned around — the line directly above the call already does
  `if (FullPath.EndsWith("/")) FullPath = FullPath.LeftChop(1)`, so the composer's `Printf("%s/%s")`
  never sees a folder ending in `/`. That `if` is a single strip, so `"/Game//"` survives it and is
  now REFUSED — the right outcome, because `FString::operator/` DOES double on a left side already
  ending in `/`, which made that input a Fatal before this change. A comment at the call site
  records that the `LeftChop` is now load-bearing. **Dead code NOT deleted** (`ExtractSavePath`
  landed recently and is another agent's work; the project rule is to report dead code, not remove
  it) — flagged here so whoever owns the cleanup can decide whether the macro, the helper and the
  `Path` slot should go.
  **`BlueprintTypeDefinitionHandler.cpp:497` — classified "no guard at all"; that verdict is WRONG,
  the site was already guarded transitively.** Both callers of `CreateAssetPackage`
  (`CreateUserDefinedStructAsset`, `CreateUserDefinedEnumAsset`) are reached only from
  `blueprint.create_struct` / `blueprint.create_enum`, and both handlers run `ParseAssetPath` ->
  `NormalizeAssetPath` (`Utils/AssetUtils.cpp:51-129`) first, which sets `bIsValid` only on a path
  `FPackageName::IsValidLongPackageName` has already accepted — via either its direct check (`:86`)
  or the fallback-root loop, which gates on the same call (`:110`). So a `//` path has always been
  refused `INVALID_ASSET_PATH` here and no editor could die through this site. An explicit
  `IsValidLongPackageName` check was still added INSIDE `CreateAssetPackage`, logged at `Warning`
  (never `Error`) and returning `nullptr`: the invariant is enforced two call frames up and nothing
  at the `CreatePackage` line said so, and the cost of a future caller composing its own path is
  process death rather than a bug. It is labelled a backstop in the code; the reachable refusal is
  unchanged.
  **Error codes: all three files checked independently and none was flipped.** All three reference
  `ErrorCodes::` ZERO times, so each is raw-literal-only and a raw `TEXT("...")` is the correct
  spelling in each (`RegistryAdoptingFilesUseConstantsOnly` is per-file). Every code emitted
  (`INVALID_ARGUMENT`, plus the pre-existing `INVALID_ASSET_PATH` / `ALREADY_EXISTS` /
  `CREATE_FAILED` the tests assert on) is already registered in `Handlers/ErrorCodes.h`; no new
  code, that header untouched.
  **Regression coverage: two files, three new leaf ids.**
  `Tests/Gameplay/TestGasCreateAbilitySetPathSafety.cpp` ->
  `PinWright.gas.create_ability_set.SetPathIsCheckedBeforePackageCreation`;
  `Tests/Blueprint/TestBlueprintCreateTypePathSafety.cpp` ->
  `PinWright.blueprint.create_enum.PathIsCheckedBeforePackageCreation` and
  `PinWright.blueprint.create_struct.PathIsCheckedBeforePackageCreation`. `check_test_ids.py` CLEAN
  (4854 ids, no dot-prefix collisions, no duplicates); `check_test_skips.py` CLEAN.
  **Fatal-unreachability, stated per file because the argument differs and is weaker at GAS than the
  foliage precedent.** At `gas.create_ability_set` the verb takes NO second argument — no mesh, no
  parent class, no attribute set — that a pre-fix build could bail on first: `setPath` is the entire
  payload, so the assignment's "pair the bad name with a well-formed other argument" construction is
  not available at this verb. The lever used instead is the handler's own already-exists branch: the
  refusal case is driven with `"/Engine/BasicShapes/Cube.Cube"`, whose `.` is in
  `INVALID_LONGPACKAGE_CHARACTERS`, so a fixed build refuses it `INVALID_ARGUMENT` above everything
  while a REVERTED build resolves it at the `LoadObject`, answers `status:"already_exists"` and
  returns — above the concatenation, having created and saved nothing, with the test red on the
  wrong outcome and the process alive. `CreatePackage` is unreachable on both builds. **The `//`
  members of the class are deliberately NOT driven at this verb, and the test header says so with a
  "do not add one" instruction**: any `//` value reaches the Fatal on a reverted build through the
  `LoadObject` and would end the suite host rather than report a red. They are covered only by the
  single `IsValidLongPackageName` call that also produces the invalid-character rejection that IS
  asserted. At `blueprint.create_*` the `//` case IS driven, safely on every build: it is refused
  above the concatenation today by `ParseAssetPath`, and if that upstream check is ever lost the new
  backstop refuses it inside `CreateAssetPackage` and the verb answers `CREATE_FAILED`, so the case
  goes red on the wrong code with the process intact — a forward guarantee rather than a
  discriminator, which the header states plainly (reverting THIS change leaves those two tests
  green, because they lock a contract the change did not alter). Both files end with a valid-input
  control that creates and saves nothing: a path naming an existing asset, answered
  `already_exists` / `ALREADY_EXISTS` from the branch below the new guard, which is the proof a
  well-formed path got past it. Both controls are gated on that fixture actually being present
  (`PinWrightTestSkip::SkipAssertions`, reason `basic-shapes-cube-fixture-absent`) — without the
  gate a well-formed path would run on into a real create inside Engine content; the GAS refusal
  case shares the gate because it is also what gives that case its reverted-build bail.
  `GAS_NOT_AVAILABLE` is skipped through the same emitter, never passed. Every refusal path carries
  a fresh GUID leaf, because `NormalizeAssetPath` retries the leaf segment under `/Game`, `/Engine`
  and `/Script` and ACCEPTS the rewrite if such a package exists, which could otherwise turn a
  refusal case into a success on some host. No readback with `UEditorAssetLibrary::DoesAssetExist`
  on a malformed path: the engine logs `DoesPackageExist called on PackageName that will always
  return false` at `Warning` (`PackageName.cpp:2415`) and `bElevateLogWarningsToErrors` defaults
  true, so that readback would fail the test it was meant to strengthen.
  **On the `..` correction:** covered at all three sites without a special case, because `.` is in
  both `INVALID_OBJECTNAME_CHARACTERS` and `INVALID_LONGPACKAGE_CHARACTERS`, so the engine calls
  used here reject `..` before it can reach the second Fatal at `UObjectGlobals.cpp:1118`.
  **On the folder-half correction (an interior `//` in the FOLDER is its own one-argument kill):**
  all three guards run `FPackageName::IsValidLongPackageName` over the FINAL string handed to
  `CreatePackage`, never over a bare name, so a folder-side `//` is refused at every one of them,
  and no hand-rolled character filter was introduced anywhere in this change. `gas.create_ability_set`
  validates the whole `setPath` and nothing mutates it between the check and
  `CreatePackage(*PackageName)`; `CreateGameFrameworkBlueprint` validates the COMPOSED
  `<FullPath>/<Name>` through `PinWrightComposeAssetPackagePath` — which is exactly why the guard
  was NOT put in `FCommonParams::ExtractSavePath`, a function that never sees the name half (its
  `SanitizeProjectRelativePath` does collapse `//` in a loop, but the create helper's own
  `/Content/`->`/Game/` normalization does not, so only the composed check is a guarantee);
  `CreateAssetPackage` validates the finished `NormalizedPath` at the `CreatePackage` line itself.
  **On the LoadObject-placement correction: re-checked at all three, and one of them needed it.**
  `gas.create_ability_set` is that one and it was already handled — its `already_exists`
  `LoadObject` sits between the guard and `CreatePackage`, so the guard was deliberately placed
  above BOTH (`IsValidLongPackageName` at `:2448`, `LoadObject` at `:2469`, `CreatePackage` at
  `:2479`), with a code comment forbidding the move back down. `CreateGameFrameworkBlueprint`
  performs no load of the composed path at all. `blueprint.create_enum` / `create_struct` probe
  existence with `UEditorAssetLibrary::DoesAssetExist`, which reads the asset registry and does not
  load; their only `LoadAsset` (`LoadAssetByRequestPath`) belongs to the EDIT verbs and runs after
  `ParseAssetPath` has already validated the path.
  **Adjacent Critical finding, NOT fixed here and NOT part of this ticket's enumeration — it wants
  its own ticket.** The same `Create=true` door exists in `GameFrameworkHandler.cpp` on a path that
  is not a `CreatePackage` site and so was never swept: `game_framework.configure_spectating` reads
  `spectatorClass` with a bare `Ctx.GetString` (`:794`) and hands it straight to
  `GameFrameworkHelpers::LoadClassFromPath` (`:149`), which calls `LoadClass<UObject>` and
  `StaticLoadObject` — `ResolveName2(..., Create=true)` -> `CreatePackage(*PartialName)`. A
  `//`-bearing `spectatorClass` therefore kills the editor from a LIVE verb without any direct
  `CreatePackage` in the plugin being involved. Left alone because it is outside the three assigned
  sites and outside this ticket's direct-call enumeration; the enumeration itself is what needs
  widening, since every unvalidated caller path reaching a LOAD is the same Critical.
  **Doc:** one `### gas.create_ability_set` section appended to `Docs/wiki-src/gas.md` — placed
  after `## See also` because it introduces that file's FIRST `###` and every `##` below one stops
  rendering. No doc change for the other two: `blueprint.create_*` behaviour is unchanged, and the
  game-framework site has no verb to document. Not compiled and not run per instruction; the
  orchestrator builds after the wave.
- `#11-material-texture-cluster-twelve-sites` `OPEN` developer — Closed the MATERIAL / TEXTURE
  cluster: twelve sites, all in the "no guard at all" class, verdict re-verified in source rather
  than trusted. Line numbers re-derived on the shared tree and every one matched the ticket exactly
  at pick time (they moved during the edits). By verb:
  `Handlers/Material/MaterialAuthoringHandler.cpp` `:1797` `create_material_function`, `:2057`
  `create_material_instance`, `:3048` `create_landscape_material`, `:3087` `create_decal_material`,
  `:3126` `create_post_process_material`, `:3168` `add_landscape_layer`, `:4036`
  `create_material_layer`, `:4094` `create_material_layer_blend`;
  `Handlers/Material/MaterialParameterCollectionHandler.cpp` `:217` `create_parameter_collection`;
  `Handlers/Material/TextureHandler.cpp` `:104` the shared `CreateEmptyTexture` helper (12 callers),
  `:2475` `channel_extract`, `:2666` `create_render_target`.
  **Yes, a shared guard covered seven of the eight in MaterialAuthoringHandler.cpp.** Seven ran one
  idiom — `MaterialCreatePathParamUtils::ResolveCreateNameAndFolder(...)` then `Path / Name`
  composed inline at the `CreatePackage` line — and every one of them wants the same rule, so a new
  `ResolveCreateAssetPackagePath` in that same header does the resolve AND the compose and is the
  single edit point for all seven. The eighth, `add_landscape_layer`, genuinely differs: its slots
  are `layerName` + `path` read straight off `Ctx`, it never touches
  `MaterialCreatePathParamUtils`, and routing it through a resolver built around the
  `name`/`assetName`/`assetPath` alias set would have changed its wire contract — it calls
  `PinWrightComposeAssetPackagePath` directly instead, as do the MPC and the three texture sites.
  **`create_material` (`:698`) was left alone and re-verified as genuinely guarded** — it runs
  `ValidateAssetCreationPath` plus an explicit `FPackageName::IsValidLongPackageName(..., true)`
  immediately above its `CreatePackage`; it is in the ticket's 19, correctly.
  **Trailing-slash over-refusal, from the sibling correction, handled per site rather than
  assumed.** `PinWrightComposeAssetPackagePath` joins with `Printf("%s/%s")`, which does not pop a
  terminator the way `FString::operator/` (PathAppend) does, so `path: "/Game/Materials/"` — which
  works today at all nine folder+name sites — would have started composing `//` and being refused.
  A new `MaterialCreatePathParamUtils::TrimTrailingFolderSeparator` is applied inside
  `ResolveCreateAssetPackagePath` (covering the seven) and at `add_landscape_layer` and
  `create_parameter_collection`. The three TextureHandler sites need no trim and that was checked,
  not reasoned around: `CreateEmptyTexture` is handed `NormalizeTexturePath(PackagePath)`, whose
  `while (EndsWith("/")) LeftChopInline(1)` provably strips them; `channel_extract`'s `outputPath`
  and `create_render_target`'s `path` are both `NormalizeTexturePath`-ed at read time, and the
  `renderTargetPath` split cuts at the last `/` so it never leaves one.
  **The FOLDER half is live here too, and nothing in this cluster collapses an interior `//`.** The
  material verbs run no folder sanitiser at all; `NormalizeTexturePath` maps `/Content` to `/Game`,
  flips backslashes and trims TRAILING slashes only. All twelve guards therefore validate the
  COMPOSED path (`FPackageName::IsValidLongPackageName`) and not merely the bare name, which is
  what covers `path: "/Game//Materials"` with a perfectly bare `name`; a hand-rolled name-only
  character filter would not have. The regression test drives that case explicitly.
  **`SanitizeAssetName` is NOT a guard, which is why `texture.create_noise_texture` was reachable
  despite having one.** Its invalid-character list (`Utils/PathUtils.cpp:261`) contains no `/`, so
  `name: "a//b"` passed it unchanged into `CreateEmptyTexture`. That helper also carries a SECOND
  Fatal above `CreatePackage`: `FPackageName::LongPackageNameToFilename`
  (`PackageName.cpp:1437-1445`) logs Fatal on a path that maps to no mount root, and its result
  (`PackageFileName`) is dead. The guard sits above both; the dead local was left in place rather
  than swept.
  **Guard PLACEMENT re-checked against the `LoadObject` -> `ResolveName2(Create=true)` ->
  `CreatePackage` door (`UObjectGlobals.cpp:1427` / `:1310`), and no site needed moving.** The
  seven resolver sites have the guard as the handler's FIRST statement, so nothing precedes it at
  all. `add_landscape_layer` and `create_parameter_collection` read their two params and guard
  immediately. `CreateEmptyTexture` guards on its first line, and its twelve callers load only the
  SOURCE texture (`texturePath`/`assetPath`) beforehand, never the composed OUTPUT path — verified
  at each caller. `create_render_target`'s guard was deliberately placed ABOVE its
  `UEditorAssetLibrary::DoesAssetExist` existence probe (registry-only, so safe either way).
  **Two out-of-scope Fatal doors found in these files and deliberately NOT closed, because they are
  the `LoadObject` class rather than this ticket's direct-`CreatePackage` enumeration:**
  `create_material_instance`'s `LoadObject<UMaterial>(nullptr, *ParentMaterial)` on raw caller text
  (`MaterialAuthoringHandler.cpp`, immediately below the new guard), `add_landscape_layer`'s
  `LoadObject<UPhysicalMaterial>(physicalMaterialPath)`, and every `StaticLoadObject(..., *SourcePath)`
  in `TextureHandler.cpp`'s processing verbs. Each is a one-argument kill on a `//`-bearing or
  `..`-bearing path. That class is plugin-wide and wants its own ticket; flagged here rather than
  swept unilaterally mid-wave.
  **Error-code adoption checked per file and it differs.** `MaterialAuthoringHandler.cpp`,
  `MaterialParameterCollectionHandler.cpp` and `TextureHandler.cpp` reference `ErrorCodes::` zero
  times, carry 76 / 11 / 5 raw literals and are in no `PartiallyConvertedHandlerFiles` baseline, so
  they use the raw `TEXT("INVALID_ARGUMENT")` and stay non-adopting.
  `Handlers/Material/MaterialCreatePathParamUtils.h` is the opposite case — it already cites
  `ErrorCodes::ERR_MISSING_REQUIRED_PARAM` and is NOT baselined, so the new refusal there uses
  `ErrorCodes::ERR_INVALID_ARGUMENT` or `RegistryAdoptingFilesUseConstantsOnly` would have failed.
  No new code: `INVALID_ARGUMENT` was already registered (`ErrorCodes.h:538`), `Handlers/ErrorCodes.h`
  untouched. The two texture subaction sites reach the wire through
  `Response->SetStringField("errorCode", ...)`, a field `RunTextureAction` already reads
  (`TextureHandler.cpp:2719`) and nothing had ever written; `CreateEmptyTexture` cannot answer a
  request at all (it returns `UTexture2D*` to 12 callers), so its refusal logs the engine reason at
  **Warning** — never Error — and the caller's existing "Failed to create texture" is what the wire
  sees. Threading an `OutError` through all 12 callers was considered and rejected as churn beyond
  the guard.
  **Regression test** `Tests/Material/TestMaterialCreateNamePathSafety.cpp`, two new leaf ids
  `PinWright.material.authoring.create_material_instance.{NameCarryingAPathIsRefused,ValidNameStillCreates}`
  (`check_test_ids.py` re-run on the shared tree: CLEAN, 4855 ids).
  **Fatal-unreachability.** `create_material_instance` is the ONLY verb in this cluster with a
  second required argument resolved above the concatenation, so it is the only one whose reverted
  build can be shown not to reach the Fatal — every bad name (and the bad-folder case) is paired
  with a well-formed `parentMaterial` naming no asset (fresh GUID under `/Game/PinWrightMissing/`).
  Fixed build: the compose check is the handler's first statement and answers `INVALID_ARGUMENT`.
  Reverted build: `LoadObject<UMaterial>` sits above `Path / Name` and answers `ASSET_NOT_FOUND`,
  so the test goes red on the wrong code with the process alive. `CreatePackage` is unreachable on
  both, and both the handler and the shared resolver carry comments forbidding the reordering that
  would break it. That fixture path is itself constrained and the header says so: the `LoadObject`
  it relies on reaches `CreatePackage` through `ResolveName2`, so it must stay mounted,
  single-slashed and dot-free (`EGuidFormats::Digits` is hex only). Six bad names (`MI_a//b`,
  rooted path, interior slash, backslash, `../Escape`, trailing slash) plus one bad FOLDER
  (`/Game//Materials` with a bare name); `..` is present because of the second Fatal, and the
  shared helper covers it because `INVALID_OBJECTNAME_CHARACTERS`
  (`Core/Public/UObject/NameTypes.h:191`) contains `.` and `:` as well as `/`. Two controls: a bare
  name + absent parent asserting `ASSET_NOT_FOUND` (proof the refusal is not blanket), and a full
  valid-input create read back from the engine, which also pins that a folder ending in `/` is
  still ACCEPTED — without that case `TrimTrailingFolderSeparator` could be deleted and everything
  else here would still pass.
  **The other eleven sites are deliberately NOT driven and that is stated rather than implied:**
  their only meaningful input IS the path (a name and a folder, nothing else), so nothing bails
  above the concatenation and a bad name on a reverted build would reach the Fatal and take the
  suite host with it. They are covered structurally — the seven other `create_*` verbs run the
  exact code path this test asserts, and the four direct callers use
  `PinWrightComposeAssetPackagePath`, whose two engine rules are already asserted by
  `TestFoliageAddTypeNamePathSafety` / `TestLandscapeCreateGrassTypeNamePathSafety`. Covering them
  behaviourally would need a second pre-concatenation bail added to each verb, which is a bigger
  change than the guard.
  **Docs:** one `##` section above the first `###` in `Docs/wiki-src/material.authoring.md`
  (listing all ten affected verbs, including `create_parameter_collection` and
  `add_landscape_layer`, which register into `material.authoring` from other files) and one in
  `Docs/wiki-src/texture.md`, which also records the `SanitizeAssetName` gap and the split between
  the wire-visible `INVALID_ARGUMENT` refusals and the log-only one. Not compiled and not run per
  instruction; the orchestrator builds after the wave.
- `#12-niagara-render-sequencer-four-sites` `OPEN` developer — Closed the four sites in
  `Handlers/Niagara/NiagaraHandler.cpp`, `Handlers/Render/RenderHandler.cpp` and
  `Handlers/Sequencer/SequencerBakeHandler.cpp`, plus one door the ticket does not enumerate
  (below). Line numbers re-derived and unchanged from the ticket (571, 638, 281, 620); none of the
  three files was touched by another agent while the work ran.
  **`NiagaraHandler.cpp:571` (`niagara.create_system`) and `:638` (`niagara.create_emitter`) —
  classification CONFIRMED ("no guard at all").** Both read `name` and `savePath` raw off the wire,
  appended a `/` if absent, concatenated, and ran the result through
  `FPackageName::ObjectPathToPackageName` — which trims at a `.`/`:` and validates nothing — before
  `CreatePackage`. Guard: `PinWrightComposeAssetPackagePath` (folder+name shape), with
  `PackagePath.RemoveFromEnd(TEXT("/"))` FIRST so the trailing slash the old `EndsWith` branch
  tolerated is not newly over-refused by the helper's `Printf("%s/%s")`. The now-inert
  `ObjectPathToPackageName` call was dropped rather than kept, and the comment says why: `.` and `:`
  are refused outright by `INVALID_OBJECTNAME_CHARACTERS`.
  **`RenderHandler.cpp:281` (`render.create_render_target`) — classification CONFIRMED.**
  `packagePath / name`, both optional and both raw, straight into `CreatePackage` with no post-call
  null check at all. Same guard, same `RemoveFromEnd` first — `operator/` absorbed a trailing slash
  here, so without the trim the helper would have started refusing `packagePath: "/Game/RT/"`.
  Edit confined to the `CreatePackage` path; nothing in the capture / exposure / subject-region /
  view-distance work in that file was touched.
  **`SequencerBakeHandler.cpp:620` (`sequencer.export_anim_sequence`) — classification DISPUTED.
  The Fatal was already unreachable there.** `outAssetPath` went through `NormalizeAssetPath`
  (`Utils/AssetUtils.cpp:51`), which sets `bIsValid` only after its own
  `FPackageName::IsValidLongPackageName` call — on BOTH of its success branches, read line by line —
  and the handler returned `INVALID_PATH` on `!bIsValid` above the composition. So this site was
  guarded transitively, not "no guard at all"; the sweep counted `NormalizeAssetPath` as
  not-a-guard. Two things WERE genuinely wrong and are fixed: the invariant was an undocumented
  postcondition of a shared exported Utils helper with several callers and a fallback branch that
  substitutes a DIFFERENT path than the caller named, and the check sat BELOW the sequence and
  binding resolution where no regression test could reach it without a live sequence fixture. The
  whole `outAssetPath` block is now the first thing the handler does, and the refusal is one `if`
  with two clauses — `!Normalized.bIsValid || !IsValidLongPackageName(Normalized.Path,
  readOnlyRoots=true, &Reason)` — so the precondition `CreatePackage` needs is asserted in the file
  that makes the call without an adjacent duplicate check. Wire text for the pre-existing failure
  mode is unchanged.
  **NOT ON THE TICKET, FOUND WHILE RE-CHECKING GUARD PLACEMENT AND FIXED IN THE SAME FILE:
  `BakeResolveSequence` (`SequencerBakeHandler.cpp:114`) is a second door to the same Fatal.**
  `StaticLoadObjectInternal` calls `ResolveName2(..., Create=true)` (`UObjectGlobals.cpp:1427`) and
  `ResolveName2` calls `CreatePackage` on the partial name (`:1310`), so its `LoadObject` on the
  RAW caller `sequence`/`path` argument was a one-argument kill reachable from BOTH
  `sequencer.export_anim_sequence` and `sequencer.bake_to_controlrig` — a verb whose enumerated
  `CreatePackage` guard would have been worthless, because the process is gone before the guard
  runs. It now checks `IsValidLongPackageName(ObjectPathToPackageName(Path), readOnlyRoots=true)`
  before the load. Nothing that worked is newly refused: a path this rejects could not have named a
  loadable sequence, and the answer is the `SEQUENCE_NOT_FOUND` an unresolvable path already gave.
  `FindObject` is safe here (`Create=false`, `:620`) and was not touched. The three enumerated
  sites were re-checked against this: neither niagara verb nor `render.create_render_target` does
  any load on the composed path, and `export_anim_sequence`'s `DoesAssetExist` / `LoadAsset`
  overwrite probe sits BELOW the hoisted guard.
  **Error-code adoption checked per file.** `NiagaraHandler.cpp` cites `ErrorCodes::` ZERO times and
  carries 30 raw `SendError(TEXT(...))`, so both niagara refusals use raw `TEXT("INVALID_ARGUMENT")`
  and the file stays non-adopting (adding one constant there would fail
  `RegistryAdoptingFilesUseConstantsOnly`). `RenderHandler.cpp` (42 `ErrorCodes::`, 0 raw) and
  `SequencerBakeHandler.cpp` (23, 0) are adopted, so they use `ErrorCodes::ERR_INVALID_ARGUMENT` /
  `ERR_INVALID_PATH`. No new codes; both were already registered (`ErrorCodes.h:538`, `:614`).
  Refusals send only — no `UE_LOG`, matching the shipped Foliage/Landscape sites.
  **Regression coverage — three new files, four new leaf ids.**
  `Tests/Niagara/TestNiagaraCreatePathSafety.cpp`
  (`PinWright.niagara.create_system.PathIsGuardedBeforeCreatePackage`,
  `PinWright.niagara.create_emitter.PathIsGuardedBeforeCreatePackage`),
  `Tests/Render/TestRenderTargetCreatePathSafety.cpp`
  (`PinWright.render.create_render_target.PathIsGuardedBeforeCreatePackage`),
  `Tests/Sequencer/TestSequencerExportAnimSequencePathSafety.cpp`
  (`PinWright.Sequencer.ExportAnimSequence.OutAssetPathIsGuardedBeforeCreatePackage`).
  `check_test_ids.py` CLEAN (4855 ids, no dot-prefix collisions, no duplicates);
  `check_test_skips.py` CLEAN.
  **THE FOLDER IS COVERED AS A KILL IN ITS OWN RIGHT, not just the name.** The composer assertions
  drive six bad FOLDERS against a perfectly bare leaf (`/Game//FX`, `//Game/FX`, `/Game/FX//Sub`,
  `/Game/../Escape`, empty, unmounted root) precisely because a folder normalizer that trims
  trailing slashes and maps aliases does NOT collapse an interior `//`, so a name-only character
  check passes every one of them through to the Fatal. The sequencer wire test drives the folder
  shape too, both at the folder/leaf boundary (`/Game/PinWrightTests//<leaf>`) and buried inside the
  folder (`/Game//PinWrightTests/<leaf>`). Both guards check the COMPOSED path, so both halves are
  refused.
  **Fatal-unreachability, and it needed TWO different constructions.** The sequencer verb takes the
  foliage/landscape construction unchanged: every bad `outAssetPath` is paired with a `sequence`
  that is a well-formed long package name naming NO asset (fresh GUID under
  `/Game/PinWrightMissing/`), so a fixed build answers `INVALID_PATH` from the hoisted block while a
  reverted build (block deleted, or moved back below `BakeResolveSequence` where it used to sit)
  answers `SEQUENCE_NOT_FOUND` — above the `CreatePackage` — and the `TestEqual` goes red with the
  process alive. That absent path is itself `//`-free, so it is safe through the new
  `BakeResolveSequence` check on a build without it. The control drives a well-formed path with the
  same absent sequence and requires `SEQUENCE_NOT_FOUND`, which proves the guard is not blanket AND
  that it sits above the sequence resolution.
  **The three niagara/render sites CANNOT take that construction, and no payload pretends
  otherwise.** Between `savePath`/`packagePath` and `CreatePackage` a reverted build passes nothing
  but `IsModuleLoaded("Niagara")` (true on every host where the file compiles) or nothing at all —
  there is no second argument to bail on, so there is NO wire payload carrying a `//` that a
  reverted build survives, and none is sent. Coverage is split instead: the whole fatal class (bad
  NAMES — rooted path, embedded `a//b`, interior slash, backslash, `../Escape`, trailing slash,
  empty — and the six bad FOLDERS above) is asserted against `PinWrightComposeAssetPackagePath`
  DIRECTLY, as a pure function whose call graph contains no `CreatePackage` on any build; and the
  wire assertion proves the handler routes through that guard using the one refusal class whose
  reverted-build behaviour is provably harmless — an UNMOUNTED package root, which composes a path
  with no `//` and no invalid characters, so a reverted build hands `CreatePackage` a string it
  accepts, builds under a clean `FName`, and answers success (red on `INVALID_ARGUMENT`, process
  alive). Routing is pinned on the composer's own message shape (the composed candidate plus the
  engine's verbatim "not a valid package path" reason), which nothing else in either handler can
  emit. Each file carries a header paragraph saying do NOT strengthen this by putting a
  slash-bearing name on the wire. Valid-input controls at the wire assert
  `errorCode != INVALID_ARGUMENT` on a bare name under a mounted folder, so they measure the guard
  rather than the Niagara factory / render-target construction, and both clean up.
  **The `BakeResolveSequence` guard ships WITHOUT a discriminating test, deliberately, and the test
  file records why.** No wire payload separates a build carrying it from one without it: the shapes
  that would discriminate (a `//` in `sequence`) end the process on the build without it, and the
  shapes that are safe to send (an unmounted root) answer `SEQUENCE_NOT_FOUND` on both. A test that
  cannot fail is worse than none and a test that can kill the host is worse than both, so the guard
  carries its reasoning at the call site instead. Same shape as `#8`'s `skeleton.create_skeleton`.
  **Confirmed in engine source rather than assumed** (`C:/UE_5.8/Engine/Source/`):
  `IsValidTextForLongPackageName` rejects `//`, a trailing slash, a missing leading slash, len<4 and
  `INVALID_LONGPACKAGE_CHARACTERS` (`PackageName.cpp:1681-1712`); `INVALID_LONGPACKAGE_CHARACTERS`
  contains `.` and `\` while `INVALID_OBJECTNAME_CHARACTERS` contains `/`, `.`, `:` and space but
  NOT `\` (`NameTypes.h:191-197`), so both halves of the composer are load-bearing; and
  `FName::IsValidXName` returns TRUE for an empty string (`UnrealNames.cpp:3932`), so the empty-name
  Fatal is caught by the composed path's trailing-slash rule, not by the name rule. That fact is
  stated at the assertion in the test rather than left implied.
  **Doc:** `### niagara.create_system` + `### niagara.create_emitter` appended to
  `Docs/wiki-src/niagara.md`, `### render.create_render_target` appended to
  `Docs/wiki-src/render.md` (both files already carried `###` sections, so no `##` is newly
  swallowed), and one paragraph added inside the existing `### sequencer.export_anim_sequence` body
  in `Docs/wiki-src/sequencer.md`. All three name the two spellings that used to be quietly repaired
  and are now refused (`.`/`:` truncation, interior `/` nesting) so the behaviour change is
  discoverable.
  **Noted, not touched:** `NormalizeAssetPath`'s fallback returns a DIFFERENT package path than the
  caller asked for whenever the bare leaf name exists under `/Game`, `/Engine` or `/Script` — with
  `overwrite:true` on `sequencer.export_anim_sequence` that is a silent rewrite of an unrelated
  asset. Separate defect, separate ticket, and `Utils/AssetUtils.cpp` was under another agent's edit
  at the time.
  Not compiled and not run per instruction; the orchestrator builds after the wave.
- `#13-audio-cluster-retyped-at-the-dispatch-boundary` `OPEN` developer — Retyped the AUDIO
  cluster's parameter declarations onto the wave's new `path` / `classref` / `filepath` vocabulary
  and folded the cluster's two safe path normalizers into the shared
  `NormalizeContentAssetPath` (`Utils/PathUtils.h`). No guard was added anywhere; the dispatch gate
  is the mechanism. **108 declarations retyped across 16 files** — 107 to `path`, 1 to `classref`
  (`audio.authoring.create_source_effect_preset`'s `effectClass`, which documents a short name, a
  class name and a `/Script/Module.Class` path, so nothing narrower than the `//` rule is correct
  on it), and **zero to `filepath`**.
  **The brief's expectation that this cluster holds the most `filepath` parameters is wrong, and
  the opposite is true: the audio cluster accepts NO disk path on any verb.** Every path-shaped
  slot here is mounted content. `audio.analysis.*` refuses a filesystem path explicitly rather
  than reading it (`AudioAnalysisHandler.cpp:432-451` rejects `.wav/.ogg/.mp3/.flac/.aif/.aiff`
  with "names a file on disk, and there is no file-reading path here by design"; `:1751-1752`
  refuses a filesystem `folder` and names `/Game/Audio` instead). The analyzer PNGs
  (`plots`) are OUTPUTS — the parameter carries plot NAMES (`waveform`, `spectrogram`,
  `constantq`) and only the response carries paths — so it stays `array|string`. WAV import lives
  in `Handlers/Asset/AssetManageHandler.cpp` (`sourcePath`), not here.
  **Not retyped, deliberately:** `nodeClassName` / `interfaceName` / `inputType` / `outputType` /
  `variableType` are MetaSound *registry* names (`UE.Sine.Audio`, `UE.OutputFormat.Mono`, `Float`,
  `WaveAsset`), not UClass references and not paths — `classref` would be a false contract in the
  wiki; `name` is a bare leaf; `targetVoices` and `stems` are arrays (see the nested gap below).
  **Normalizer fold — de-duplication, and byte-identical.** Deleted the file-static
  `NormalizeAudioPath` (`AudioAuthoringHandler.cpp`) and `PinWright::MetaSound::NormalizeAudioAssetPath`
  (`MetaSound/MetaSoundPathUtils.{h,cpp}`) and renamed **66 call sites** to `NormalizeContentAssetPath`
  — 48 in `AudioAuthoringHandler.cpp` (plus one comment) and 18 across seven MetaSound TUs. The two
  bodies were byte-equivalent to each other and to the shared one; **the only observable difference
  is the prefix of the rejection `UE_LOG(..., Warning, ...)`**, which now reads
  `NormalizeContentAssetPath:` instead of `NormalizeAudioPath:` / `MetaSound NormalizeAudioAssetPath:`.
  Reachability: `AudioAuthoringHandler.cpp` gets `Utils/PathUtils.h` through `PinWrightHelpers.h`;
  for the MetaSound TUs the include was added to `MetaSoundPathUtils.h`, which all seven already
  include for the compat macros, rather than to each `.cpp`. Net −75 lines in the cluster.
  **`EveryCreatePackageIsGuarded` checked before AND after**: `AudioAuthoringHandler.cpp` 13/13 and
  `MetaSound/MetaSoundPatchPresetHandler.cpp` 2/2, total 15 — unchanged, and
  `CountOccurrences("PackagePath = Path / Name")` is still 0. Neither edit touches either token.
  (Separately: `FAudioPackagePathGuardTrailingSlashFolderComposesTest` in the same file asserts the
  trailing-slash trim that agent A1 is deleting from `AudioPackagePathGuard.h`; that is A1's to
  reconcile, not this entry's.)
  **Two live editor-kill surfaces found in this cluster that the wave brief said did not exist.**
  The claim "the audio cluster's 49 load sites were never at risk" holds only for the sites fed by
  the two normalizers. Two more helpers normalize by hand and never call
  `SanitizeProjectRelativePath`, so they collapse no `//` and reject no `..`:
  `SoundWaveAuthoringHandler.cpp:26-36` `LoadSoundWaveByPath` (only `\Content\`→`/Game/`,
  backslashes, trailing slashes) feeding `StaticLoadObject` at `:35`, and
  `AudioSynthAuditionHandler.cpp:47-68` `PwResolveAuditionSound` (backslashes and trailing slashes
  only) feeding `StaticLoadObject` at `:67`. `audio.authoring.set_sound_wave_properties
  {assetPath: "/Game//X"}` and `audio.synth.audition {assetPath: "/Game//X"}` were both process
  death TODAY. A third shape: `AudioHandler.cpp:246/253` and `:647/654` hand
  `Ctx.GetString("attenuationPath")` / `("concurrencyPath")` straight to `LoadObject<>` with no
  normalization at all. **All four are now closed by the retype**, which is the wave's whole
  point — no guard was added, and none is needed while the declarations stand. The two hand-rolled
  normalizers were left in place: folding them into `NormalizeContentAssetPath` would newly reject
  `..` and unmounted roots on those verbs, which is a contract change the dispatch gate does not
  require.
  **Residual, and it needs agent B's nested gate, not a guard:** `AudioMusicHandler.cpp:1744`
  loads `Stem.AssetPath` from the nested `stems[].assetPath` of `audio.music.create_interactive_music`.
  The top-level type gate cannot see it. Same shape for `dialogue`'s `targetVoices` array and
  `create_sound_mix`'s `classAdjusters[].soundClass`.
  **`SoundCueDumpBuilder::NormalizeSoundCuePath` (`Handlers/Asset/SoundCueDumpBuilder.cpp:85-115`)
  — reported, NOT edited (outside this agent's files), and it CANNOT fold as-is.** Its first three
  steps (`FPaths::NormalizeFilename`, the `while (Contains("//"))` collapse, the leading-slash
  prepend) are exact duplicates of work `SanitizeProjectRelativePath` already does
  (`PathUtils.cpp:58/68/81`), so they are redundant, not different. The one real divergence is
  ORDERING: it rewrites `/Content`→`/Game` **before** the sanitizer, so `/Content/Audio/SC_X`
  survives; `NormalizeContentAssetPath` sanitizes first, and `IsValidMountPoint("/Content/...")`
  is false, so that input is refused and returns empty. Folding it silently drops a working input
  shape on `audio.authoring.describe_sound_cue` and friends.
  **Corollary worth recording about the newly shared helper:** for the same reason, the
  `/Content`→`/Game` branch inside `NormalizeContentAssetPath` (`PathUtils.cpp:117-125`) is
  **unreachable** — it runs only on a string that already passed the mount check, and no
  `/Content...` string ever does. It was dead in `NormalizeAudioPath` too, which is why this fold
  is behaviour-preserving. Removing it is a separate, tiny cleanup; whether `/Content` should be
  accepted at all is a product question, not a safety one.
  **Not cleaned up, on purpose:** `MetaSoundPathUtils.cpp` no longer uses
  `SanitizeProjectRelativePath` or `LogPinWrightSubsystem`, so its `Utils/PathUtils.h` (now
  redundant with the header's) and `PinWrightSubsystem.h` includes are orphans of this change.
  Dropping headers from a Unity-built TU while other agents edit the same blob is not worth the
  risk in this wave; left for the orchestrator's post-build pass.
  Not compiled and not run per instruction; the orchestrator builds after the wave.
- `#14-ai-gas-character-networking-declaration-retype` `OPEN` developer — Layer-1 DECLARATION RETYPE
  only for the AI / GAS / game-framework / character / networking cluster: `Handlers/AI/**`
  (`AIHandler.cpp`, `NavigationHandler.cpp`, `BehaviorTreeHandler.cpp`, `EQSHandler.cpp`,
  `StateTreeAuthoringHandler.cpp`, `BehaviorTreeDecompileHandler.cpp`),
  `Handlers/Systems/{GASHandler,GameFrameworkHandler}.cpp`, `Handlers/Character/CharacterHandler.cpp`,
  `Handlers/Networking/NetworkingHandler.cpp`. **167 `FParamSpec::Type` declarations retyped: 136
  `path`, 31 `classref`, 0 `filepath`.** No guard, no handler code, no test, no doc, no error code —
  the dispatch gate (`Handlers/ParamTypeCheck.h`) is the mechanism and the declaration is its input.
  Per file, `path`/`classref`: AIHandler 40/5, NavigationHandler 3/6, BehaviorTreeHandler 4/6 (+1
  helper), EQSHandler 6/3, StateTreeAuthoringHandler 5/4, BehaviorTreeDecompileHandler 1/0,
  GASHandler 31/3 (+2 raw specs), GameFrameworkHandler 8/2, CharacterHandler 18/0,
  NetworkingHandler 19/0.
  **`spectatorClass` is declared `classref`** (`GameFrameworkHandler.cpp:780`), as asked.
  **THREE DECLARATIONS IN THIS CLUSTER DO NOT USE THE `RPC_PARAM_*` MACROS and a macro-shaped lint
  regex will not see them.** `BehaviorTreeHandler.cpp:66` builds its spec with
  `ParamAliasUtils::MakeAliasParamSpec(TEXT("assetPath"), TEXT("path"), ...)` inside
  `BTAssetPathParamReq()`, one helper covering SEVEN `behavior_tree.*` verbs at `:666, :856, :870,
  :884, :955, :988, :1021`; and `GASHandler.cpp:1541` / `:1673` are brace-initialised
  `FParamSpec{TEXT("attribute"), TEXT("classref"), ...}` written out longhand because they carry an
  `attributeName` alias. All three are retyped here, but whoever writes the Layer-1 ratchet must
  match `FParamSpec` construction generally, not `RPC_PARAM_` — otherwise it reports this cluster
  clean while eight verbs are undeclared.
  **Four live `//` editor kills found in this cluster that the ticket's direct-`CreatePackage`
  enumeration does not list**, each reached from raw caller text with nothing argument-driven above
  it. They are recorded because they are evidence for the wave's premise that `CreatePackage` is not
  the door that matters: (a) `game_framework.configure_spectating` reads `spectatorClass` with a bare
  `Ctx.GetString` and hands it straight to `LoadClassFromPath` -> `LoadClass` -> `StaticLoadObject`
  (`GameFrameworkHandler.cpp:794-797`); (b) the SEVEN `GF_CREATE_CLASS_HANDLER` verbs read
  `parentClass` into the same `LoadClassFromPath` (`GameFrameworkHandler.cpp:270`) — a second door in
  the same file, and the more reachable one; (c) `behavior_tree.create_{task,service,decorator}_blueprint`
  read `parentClass` into `ResolveBTBlueprintParentClass` (`BehaviorTreeHandler.cpp:196-217`), whose
  second attempt is `LoadClass<UObject>(nullptr, *Trimmed)` on the untouched caller string; (d)
  `gas.add_effect_modifier` / `gas.set_modifier_attribute` read `attribute`, and
  `ResolveGameplayAttributeFromSpec` (`GASHandler.cpp:601-612`) splits it on the LAST `.` and calls
  `LoadObject<UClass>` on the left half — `//` carries no dot, so it survives the split intact and
  the split is not a sanitiser. None of these four is a `CreatePackage` call site; all four are now
  refused at dispatch.
  **Classifications that a find-replace on the parameter NAME would have got wrong**, each read at
  the call site rather than inferred from the spelling. Typed `classref` despite ending in `Path`:
  `costEffectPath`, `cooldownEffectPath` (`GASHandler.cpp:1166`/`:1220`) name a GameplayEffect CLASS
  and go through `ResolveGameplayEffectClassArg` / `ResolveGasSubclassArg`, never an asset load.
  Typed `path` despite containing `Class`: `executionClassPath` (`:2976`), because `:2987` does
  `LoadObject<UBlueprint>` on it — it is a blueprint asset path. Typed `path`: `enumPath` (`:2754`),
  an object path handed to `LoadObject<UEnum>` (`:2798`). Typed `classref` despite a `*Type` name:
  `nodeType` (`BehaviorTreeHandler.cpp:667`, resolved by `ResolveClassByName` at `:771`) and
  `generatorType` / `testType` / `contextClass` (`EQSHandler.cpp:820`/`:832`/`:847`, resolved by
  `ResolveClassByPathOrName` at `:65-82`) each accept EITHER a built-in vocabulary word OR a class
  path, which is exactly the shape `classref` was added for.
  **One judgement call, flagged rather than buried:** `evaluatorClass`, `taskClass`, `conditionClass`
  and `payloadStruct` (`StateTreeAuthoringHandler.cpp:510`/`:556`/`:612`/`:704`) are **UScriptStruct**
  references, not UClass — they resolve through `ResolveUScriptStruct` (`:124`, `:748`), the
  chokepoint the plan lists separately from `ResolveUClass`. They are typed `classref` because the
  vocabulary has no struct spelling, `classref`'s `//`-only rule is precisely correct for them, and
  three of the four are name-matched by any `*Class` ratchet anyway. If a `structref` token is ever
  added these four move. `attribute` (`GASHandler.cpp:1541`/`:1673`) is the same kind of
  approximation in the other direction: its value is a class path with a member name appended after a
  final `.`, so `classref` also refuses a `//` appearing only in the member half — nonsense input, but
  a wider refusal than the load strictly needs.
  **Deliberately left `string`, because a `*Path` sweep would have mis-declared them.**
  `state_tree.add_binding`'s `sourcePath` and `targetPath` (`StateTreeAuthoringHandler.cpp:806-807`)
  are PROPERTY-BINDING EXPRESSIONS (`nodeId:property`, `nodeName.property`, or a bare property paired
  with `sourceId`/`targetId`), not paths of any kind; typing them `path` would publish a rule that
  does not apply to the slot. AIHandler's `compositeType` (`:658`), `taskType` (`:675`),
  `decoratorType` (`:692`) and `serviceType` (`:703`) are closed node-kind vocabularies on four verbs
  that are already disabled and answer `DEPRECATED_HANDLER` unconditionally, resolving nothing —
  note the near-collision, because the LIVE deprecated aliases in the same file (`generatorType`
  `:728`, `contextType` `:738`, `testType` `:748`) route into the real `PinWrightEQS` handlers and DID
  become `classref`. `attributeName` at `GASHandler.cpp:846`/`:916` is a bare attribute name and stays
  `string`; the identically spelled `attributeName` at `:1543`/`:1675` is an ALIAS on the `attribute`
  slot and inherits `classref` through the gate's alias resolution, so the two spellings are typed
  differently on purpose.
  **`filepath` count is zero, and that was measured rather than assumed.** All ten files were grepped
  for disk-path shapes (`file`, `directory`, `folder`, `export`, `import`, `output`, `destination`,
  `absolute`, file extensions) with no hit. The one trap is `savePath` (`BehaviorTreeHandler.cpp:551`,
  `:631`, `:643`, `:655`), which reads as a disk name and is documented "Folder path (default /Game)"
  — a CONTENT folder. It is `path`. A cluster agent applying the plan's `*Path`-is-disk heuristic
  without reading would have turned four working verbs into UNC-shaped `filepath` slots.
  **What was skipped, and why.** `BehaviorTreeDecompileHandler.cpp:51` is this cluster's ONLY
  `Ctx.RequireAssetPath` caller (whole-cluster grep) — its declaration is still retyped to `path`
  because the declaration is the published contract, but no guard was added: `RequireAssetPath`
  already runs `SanitizeProjectRelativePath` AND re-checks with `IsValidAssetPath`. Everything
  reaching `ResolveUClass` / `ResolveClassByName` / `ResolveUScriptStruct` was left to the
  chokepoint-guard agent; the declarations were retyped regardless, per the wave's rule that the
  dispatch gate is the primary defence and the guard is depth. No `UEditorAssetLibrary::LoadAsset`
  site in this cluster needed anything. `GameFrameworkHandler.cpp` and `NetworkingHandler.cpp` were
  touched for DECLARATIONS ONLY per the ownership split; the concurrent unification of their two
  duplicate `LoadBlueprintFromPath` copies into `Handlers/Blueprint/BlueprintPathLoad.h` was present
  in the working tree and is untouched here.
  **Not covered by this retype, and it needs the nested gate:** `GASHandler.cpp:2977`
  (`gas.set_execution_capture`) declares `captures` as an `array` whose per-entry `attribute` values
  carry the same `AttributeSetClassPath_C.AttrName` shape and reach the same
  `LoadObject<UClass>` at `:601-612`. The top-level type gate never sees a nested value, so the
  `//` refusal does not reach it. This is a concrete candidate for the `NestedParamKeyCheck.h`
  extension the plan describes.
  **Error-code trap checked per file and cleared:** `grep -c "ErrorCodes::"` is **0 at HEAD and 0
  now** in all five files that carry raw literals (`GameFrameworkHandler.cpp`,
  `NetworkingHandler.cpp`, `AIHandler.cpp`, `GASHandler.cpp`, `CharacterHandler.cpp`). No
  `ErrorCodes::` reference was introduced anywhere, so no non-adopting file was flipped into
  `RegistryAdoptingFilesUseConstantsOnly`'s scope; `Handlers/ErrorCodes.h` is untouched. This
  retype emits no code at all, so no refusal message and no error code changed. No test added, so
  `check_test_ids.py` was not re-run. No doc change: a `Type` retype alters no verb behaviour, and
  the wiki renders that same field, so every namespace page picks the new spelling up on the next
  regeneration.
  **BLOCKING CROSS-AGENT GAP, verified in source rather than predicted.** `ParamSpec.h` and
  `ParamTypeCheck.h` have landed the three new tokens, but the PRE-EXISTING registry-wide vocabulary
  test has not been extended: `PinWright.infra.contract.ParamTypes.ValidTypeNames`
  (`Tests/Infra/TestContractConsistency.cpp:184`) builds `ValidAtomicTypes` at `:192` from the
  original eight tokens only, and `IsSupportedTypeExpr` (`:203`) hard-fails any part not in that set.
  Until `path`, `classref` and `filepath` are added there, that test goes RED on all 167 declarations
  below AND on every other retype cluster in this wave — it iterates
  `FAutoRegisterHandler::GetPendingRegistrations()` plugin-wide, so this is not a per-file failure.
  That file is outside this agent's ownership and was deliberately not edited. Stated here so a
  partial landing is diagnosable as an ordering problem rather than a bad retype.
  Not compiled and not run per instruction; the orchestrator builds after the wave.
- `#15-material-texture-cluster-retyped-and-normalizer-folded` `OPEN` developer — Layer-1
  DECLARATION RETYPE plus the Layer-3 normalizer fold for the MATERIAL / TEXTURE cluster
  (`Handlers/Material/**` less `MaterialCreatePathParamUtils.h`, `MGIR/**`,
  `Material/MaterialExpressionFactory.cpp`). **115 declarations retyped: 113 `path`, 2 `classref`,
  ZERO `filepath`** — and the zero is a measurement, not an omission. The wave brief warned that
  `sourcePath` / `outputPath` are disk paths in texture import/export verbs; **this cluster
  registers no import or export verb.** `TextureHandler.cpp`'s 26 registrations are enumerated at
  `:2791-3030` and every one is a create / set / process verb whose paths are package paths. The
  two names were read individually anyway: `texture.resize_texture`'s `sourcePath` (`:2939`) is
  fed to `SanitizeProjectRelativePath` and then `StaticLoadObject` at `TextureHandler.cpp:1500`,
  and `texture.channel_extract`'s `outputPath` (`:3017`) is handed to
  `PinWrightComposeAssetPackagePath` at `:2491` — both mounted asset paths, both `path`. The file
  includes `HAL/PlatformFileManager.h` and uses nothing from it; that orphan predates this wave and
  was left alone.
  **Where the 115 are.** 56 through `MaterialHandlerUtils::MaterialAssetPathParamReq`'s explicit
  `Type` argument (48 in `MaterialAuthoringHandler.cpp`, 8 in `MaterialGraphHandler.cpp` — the
  `assetPath`/`materialPath`/`path` alias slot); 2 through
  `MaterialExpressionClassParam{Req,Opt}` (`expressionClass`, `nodeType`) → **`classref`**, which
  is the right token and `path` would have been wrong: `material.graph.add_expression` documents
  the bare short form `Add` alongside `MaterialExpressionAdd`; 57 direct `RPC_PARAM_*`
  (`TextureHandler.cpp` 38, `MaterialAuthoringHandler.cpp` 8 — `texturePath`x3, `parentMaterial`x2,
  `functionPath`, `physicalMaterialPath`, `add_landscape_layer`'s folder `path`;
  `MaterialParameterCollectionHandler.cpp` 8; `MaterialGraphHandler.cpp` 1;
  `MGIRCompileHandler.cpp` 1; `MGIRDecompileHandler.cpp` 1).
  **One retype closes a kill path the brief did not list.** `material.compile_mgir`'s `context`
  (`MGIRCompileHandler.cpp:40`) is not path-SHAPED by name — it reads as a mode or a scope token —
  but it is a fallback asset path: `Options.Context` → `NormalizeEntryTarget`
  (`MGIR/MGIRCompiler.cpp:300-307`) → `GetOrCreateMaterial` → `LoadObject<UMaterial>(nullptr,
  *AssetPath)` (`:325`). `material.compile_mgir {context: "/Game//X"}` was process death today.
  A name-driven lint test would never have flagged it; it was found by reading the description.
  **Normalizer fold.** Deleted the file-static `NormalizeTexturePath` (`TextureHandler.cpp:57-70`)
  and renamed **19 call sites** to `NormalizeContentAssetPath`, plus 3 comment mentions in the same
  file and 2 in `Tests/Material/TestMaterialCreateNamePathSafety.cpp` (`:45`, `:154`) whose stated
  fact — "only maps /Content to /Game, flips backslashes and trims TRAILING slashes" — the deletion
  falsified. Reachability was verified, not assumed: `TextureHandler.cpp` includes
  `PinWrightHelpers.h`, which includes `Utils/PathUtils.h` at `:9`. Net −16 lines.
  **The fold is NOT byte-equivalent here, unlike the audio one, and this is the entry's one real
  contract change.** `#6`/`#13` record that `NormalizeAudioPath`'s `/Content`→`/Game` branch was
  already dead because the sanitizer ran first. `NormalizeTexturePath` ran NO sanitizer, so its
  rewrite was **live**: `path: "/Content/Textures"` composed `/Game/Textures` and worked on those
  19 sites, and now returns empty and is refused. Three facts bound the blast radius. (a) The other
  18 path-reading sites in the same file already called `SanitizeProjectRelativePath` directly and
  already refused `/Content` — the cluster was internally inconsistent and the fold makes it
  consistent, not uniformly stricter. (b) No test, no wiki page and no param default in the cluster
  uses the `/Content` spelling; the only `/Content` payload in the suite is
  `TestAudioHandlers.cpp:1367` against `decompile_sound_cue`, which routes through
  `SoundCueDumpBuilder::NormalizeSoundCuePath` and is untouched by this. (c) Every one of the 19
  sites already tests the result for emptiness and answers a refusal (`assetPath is required`,
  `name is required`, `texturePath is required`, or the composer's `INVALID_ARGUMENT`), so the
  narrowing is a refusal, never a Fatal and never a silent wrong asset. The 19 sites also **gain**
  `..`-rejection and interior-`//` collapsing they never had.
  **SKIPPED, with the ground for each.** (1) **22 `Ctx.RequireAssetPath` sites in
  `MaterialAuthoringHandler.cpp`** plus `MaterialGraphHandler.cpp:334`,
  `MaterialParameterCollectionHandler.cpp:113`, `MGIRDecompileHandler.cpp:53`, three in
  `MaterialFinders.h` and 2 in `TextureHandler.cpp` (`:2878`, `:2910`) — already sanitized AND
  re-validated (`HandlerContext.cpp:113`/`:133`); no guard added. Their DECLARATIONS were still
  retyped, because the declaration is the wiki contract and the ratchet reads names, not bodies.
  (2) **18 `SanitizeProjectRelativePath` sites in `TextureHandler.cpp`** (`:381, 568, 739, 904,
  962, 1147, 1211, 1275, 1325, 1374, 1423, 1500, 1545, 1642, 1675, 1754, 1788, 1871`) — the brief
  said 11; there are 18, and **all 18 assign the sanitized value back over the raw variable**
  (`Path = SanitizedPath` / `AssetPath = SanitizedAssetPath` / `SourceTexture = SanitizedSource`),
  which is the distinction that made the `LevelStructureHandler` reading wrong in `#2`. Checked
  one by one; none is the compute-then-discard shape. (3) **`nodeId` / `sourceNodeId` /
  `targetNodeId` / `expressionId` left `string`** — `MaterialFinders.h:47-74`
  `FindExpressionInView` is a pure in-memory scan over `GetExpressions()` comparing GUID, name,
  `GetPathName()` and parameter name. No load, no `CreatePackage`; `path` would be a false
  contract on a slot that legitimately takes a GUID. (4) **`name` / `layerName` left `string`** —
  bare leaves, and each is already validated by `PinWrightComposeAssetPackagePath`'s
  `FName::IsValidXName` half. (5) **`TextureHandler.cpp:2700`'s `DoesAssetExist`** is NOT the
  `AssetUtils.cpp:1382-1384` anti-pattern: the composer validated `FullPath` at `:2694` and the
  probe reads that same validated string, i.e. the guard is already above the `if`. Nothing to
  report there.
  **Residual, and it needs a nested-VALUE gate, which is not what agent B is building.** Three
  loads in this cluster take caller text the top-level type gate cannot see. (a)
  `material.graph.create_nodes` → `nodes[].texturePath` → `LoadObject<UTexture>`
  (`MaterialGraphHandler.cpp:587`). (b) `material.authoring.set_material_instance_parameters` and
  `create_material_instance` share one applier: `texture: {ParamName: AssetPath}` →
  `LoadObject<UTexture>` (`MaterialInstanceOverrides.h:116`). **Note the shape** —
  `create_material_instance` already declares `NestedKeys` (`scalar`/`vector`/`texture`/
  `staticSwitch`), so a nested-KEY gate is adopted there and still does not help: the paths are the
  nested VALUES, under caller-chosen parameter names. (c) `material.compile_mgir`'s entry-block
  names live inside the `text` document and are split by the compiler, so no declared type on any
  parameter can reach them; `MGIRCompiler.cpp:325` is the load. No code was emitted for any of the
  three.
  **Error-code trap avoided.** `TextureHandler.cpp` is non-adopting with raw literals and is not on
  the `PartiallyConvertedHandlerFiles` baseline; the diff adds zero `ErrorCodes::` references
  (verified by grepping the `+` side of the diff). No tests added, so
  `Content/Python/check_test_ids.py` needs no re-run. `sed -i` normalized CRLF→LF in five edited
  `.cpp` files; `.gitattributes` `text=auto` normalizes both ways in the index, so the committed
  diff is the edits alone — checked with `git diff --stat` (122 changed lines in a 4,329-line
  `MaterialAuthoringHandler.cpp`, not 4,329).
  **For agent A1 / the orchestrator, not fixed here:** `MaterialCreatePathParamUtils.h`'s two
  shared specs (`MaterialCreateNameParamReq` → `name`, `MaterialCreateFolderParamOpt` → `path`)
  still declare `string`, and they cover the destination slot of all eight
  `material.authoring.create_*` verbs. That file is A1's. If A1 retypes the folder slot to `path`,
  `ExpectInstanceFolderRefused` (`TestMaterialCreateNamePathSafety.cpp:244`, driving
  `path: "/Game//Materials"`) starts being refused at the dispatch gate instead of at the composer,
  and its asserted refusal shape must be re-checked.
  Not compiled and not run per instruction; the orchestrator builds after the wave.
