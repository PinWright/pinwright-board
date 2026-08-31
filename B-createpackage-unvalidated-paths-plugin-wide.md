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
