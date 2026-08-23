---
id: B-asset-verification-clobbers-handler-assetpath
title: "AddAssetVerification silently overwrites the handler's assetPath with the package path at ~200 call sites, contradicting its own comment"
status: IN-REVIEW
severity: High
category: bug
tags: [asset-verification, assetPath, response-contract, object-path, package-path, handler-authoring, cross-cutting]
---

# A verb's `assetPath` is not the value the verb set

`WriteMeasuredAssetVerification` writes the top-level `assetPath` field from
`ResolveVerificationAssetPath()`, which for a top-level asset deliberately
returns the bare **package** path (`Utils/AssetUtils.cpp:1346`, `:1376`) —
`/Game/Foo/SW_X`. Any handler that already set `assetPath` to the **object**
path — `/Game/Foo/SW_X.SW_X`, which is what `UObject::GetPathName()` returns —
has that value silently replaced, because `AddAssetVerification` runs after the
handler's own assignment.

The overwrite is almost certainly unintended. The function's own comment reads
*"Package name, not the assetPath above"*, which only makes sense if the author
believed they were writing a **different** field and that the handler's
`assetPath` would survive. It does not.

## Why this matters

`assetPath` is the field an agent feeds straight back into the next call. Object
path and package path are not interchangeable across the whole verb surface, so
a handler that correctly reports an object path has its response downgraded to a
form the next verb may reject — and nothing in the response signals the
substitution happened.

Scope is wide: roughly **200 `AddAssetVerification` call sites**. How many are
actually wrong depends on how many set `assetPath` themselves beforehand; that
census has not been taken and is the first task here. Verbs that never set the
field see the package path as intended and are unaffected.

There is an observable inconsistency in the shipped surface already:
`audio.music.export_stems` asserts a dotted **object** path for each created
wave's `assetPath` and passes, while
`audio.authoring.create_sound_wave_from_pcm` was returning the bare package path
for the same kind of result. Two sibling verbs, two contracts.

## Current state

`create_sound_wave_from_pcm` was fixed **locally only**, by moving its
`assetPath` assignment to after `AddAssetVerification` in both the success and
decode-failure branches (`Private/Handlers/Audio/SoundWavePcmHandler.cpp`). That
is a per-verb workaround, not the fix — every other affected verb still has the
defect, and the next handler written will reintroduce it, because the ordering
requirement is invisible at the call site.

## Suggested remedy

Per `rpc-design.md` §2, the structural fix beats the remembered one: make it
impossible for the verification helper to clobber a value the caller set.
Options, cheapest first:

1. **Don't overwrite a populated field.** Have `WriteMeasuredAssetVerification`
   write `assetPath` only when absent. Smallest diff, but leaves the ordering
   dependency latent for anyone who reads the code rather than the behaviour.
2. **Write the package path to its own field** (`packagePath`) and stop touching
   `assetPath` at all. Matches what the comment says the code was doing. Changes
   the response shape of ~200 verbs, so it needs a sweep of tests asserting
   `assetPath`.
3. **Take `assetPath` as an explicit parameter** to `AddAssetVerification`, so a
   handler states its intent instead of racing the helper.

Take the census before choosing — if most call sites never set `assetPath`,
option 2 is close to free.

## Evidence

Found while fixing the `create_sound_wave_from_pcm.RoundTrip` test failure,
whose sole assertion error was `expected /Game/…/SW_X.SW_X, got /Game/…/SW_X`.
Every decode and verification assertion in that test passed — the failure was
purely the clobbered path field. Mechanism confirmed by reading
`Utils/AssetUtils.cpp:1346,1376` and the `AddAssetVerification` write order.

## History

- `#1-filed` — **OPEN** (Reporter). Filed while driving the audio subsystem
  test suite to green. One verb patched locally as a symptom fix; the shared
  helper is untouched and is the actual defect.
- `#2-second-instance-confirmed` — **OPEN** (Reporter). A second verb hit the
  identical defect in the very next suite run:
  `PinWright.audio.synth.export.VerifiesAndIsIdempotent` failed with
  *expected `/Game/PinWrightTests/SW_SynthExport_<guid>.SW_SynthExport_<guid>`,
  got `/Game/PinWrightTests/SW_SynthExport_<guid>`* — object path downgraded to
  package path, same mechanism, different handler
  (`Handlers/Audio/AudioSynthGenerateHandler.cpp`). Two of the three
  asset-creating audio verbs were affected; the third
  (`audio.music.export_stems`) happens to write `assetPath` after verification
  and passes. That is the ordering dependency being decided by accident, which
  is the argument for the structural fix over per-verb reordering. Patched
  per-verb again to unblock the suite; the shared helper remains untouched.
- `#3-additional-contract-census` `OPEN` reporter — Additional evidence: **Adversarial review A — REFRAME.** Actuality: CONFIRMED CURRENT. Framing: the clobber is live — `WriteMeasuredAssetVerification` unconditionally writes `assetPath`, and current audio creates write `GetPathName()` before `AddAssetVerification` — but the universal package-path claim is too broad: UE's loader retries a dotless path as the package's main object, current test utilities document package-path output as the `create_*` contract, and only subobjects require the full path; the two historical audio verbs named in this ticket are absent from the current checkout. The comment `Package name, not the assetPath above` describes the measured probe's local package key, not a second response field. A read-only census finds 307 production direct calls and 47 calls with a nearby caller `assetPath` assignment (heuristic, not a semantic affected count). Proposed fix: INCOMPLETE, a shared preserve-populated-field guard is the right structural direction, but it changes current package-path outputs and needs a path-contract audit/tests (including an explicit package field if both handles are required); `packagePath` migration or per-verb reordering is not justified yet. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\AssetUtils.cpp:1339-1369`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Audio\AudioAuthoringHandler.cpp:514-517`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Assets\TestCreateSoundCueLooping.cpp:33-38`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\TestUtils.h:541-546`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\World\TestLevelHandlers.cpp:1997-2008`, `C:\UE_5.8\Engine\Source\Runtime\CoreUObject\Private\UObject\UObjectGlobals.cpp:1474-1481`. Runtime: NOT VERIFIED. Recommendation: REFRAME; narrow to loss of caller-authored `assetPath`, separate top-level/subobject behavior, then add helper regression tests and audit current consumers before editing source.
- `#4-additional-object-contracts` `OPEN` reporter — Additional evidence: **Adversarial review B — REFRAME.** Actuality: **CONFIRMED CURRENT**. Framing: I agree with A that the unconditional write and the two cited historical audio names are stale, and that `Package name, not the assetPath above` describes the persistence probe. I disagree that only subobjects need a full path: current `input.create_input_action` and `input.create_input_mapping_context` set `assetPath` from `GetPathName()` immediately before the helper, and their mapping verbs document object-path inputs; `behavior_tree.create` likewise documents a dotted object-path result. `ResolveUObjectByPath` can return the `UPackage` for a bare path, so the MetaSound loader explicitly retries `Package.AssetName`; the UE `StaticLoadObject` fallback does not make `StaticFindObject`-based consumers equivalent. The 307 production calls are real; a same-function scan finds 46 caller-authored candidates (33 explicit `GetPathName()`), not merely an audio artifact. Proposed fix: **SYSTEMIC**, define the response contract as canonical object `assetPath` plus explicit `packagePath` (or pass path intent into the helper), then audit both top-level and subobject consumers with regression tests; a preserve-if-populated guard alone leaves request-path callers and ordering hazards. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\AssetUtils.cpp:1339-1370`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Input\InputHandler.cpp:44-65`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Input\InputHandler.cpp:134-146`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Input\InputHandler.cpp:244-265`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\wiki-src\behavior_tree.md:9-28`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Audio\MetaSound\MetaSoundPathUtils.cpp:47-60`, `C:\UE_5.8\Engine\Source\Runtime\CoreUObject\Private\UObject\UObjectGlobals.cpp:1474-1481`. Runtime: NOT VERIFIED. Recommendation: REFRAME; retain High priority for the silent cross-cutting contract mismatch and schedule a path-contract census plus helper-level top-level/subobject tests before implementation.

- `#5-census-taken-structural-guard-landed` `IN-REVIEW` developer — **Census first, as the ticket asked.** A stdlib scan of `Source` counts **322** direct `AddAssetVerification` calls across 74 files (`AddAssetVerificationNested` excluded — it writes into its own nested object and cannot clobber a top-level field), of which **50 production call sites** assign `assetPath` earlier in the same handler body. About 35 of those assign `X->GetPathName()`, i.e. an object path downgraded to the bare package path for a top-level asset; the rest assign `Path / Name` (already package-shaped, no observable change) or a caller-supplied string. Concentrations: `Handlers/Audio/AudioAuthoringHandler.cpp` (12), `Handlers/Animation/*` (7), `Handlers/Blueprint/BlueprintCreationHandler.cpp` (5), `Handlers/Input/InputHandler.cpp` (4), `Handlers/Asset/AssetManageHandler.cpp` (4), `Handlers/AI/BehaviorTreeHandler.cpp` (3). **Chosen remedy: none of the three listed — a fourth, because option 1 is unsafe here.** Reviews A and B are both right and they conflict: A is right that a blanket preserve-if-populated changes current package-path outputs, B is right that object paths are genuinely required by some consumers. Blanket preserve is wrong for a reason neither named: several of the 50 sites echo a caller-supplied REQUEST string (`FinalAssetPath`, `DestinationPath`, `SafePath`), so preserving unconditionally trades a silent downgrade for a silent echo of unverified input. Implemented instead in `Utils/AssetUtils.cpp` (`WriteMeasuredAssetVerification`): the handler's `assetPath` is KEPT when it denotes the same PACKAGE as the object actually verified, and REPLACED otherwise, with `requestedAssetPath` + `assetPathSubstituted:true` recording it — a substitution the caller cannot see is the defect, not the substitution itself. Package path, object path and sub-object path for one asset all reduce to the same package name via `FPackageName::ObjectPathToPackageName`, so every honest spelling survives and option 2's ~200-verb response-shape churn is avoided. `packageName` is now always emitted under its own key, which is what the `existsAfter` comment ("Package name, not the assetPath above") always implied; **`packagePath` was deliberately NOT used** — `asset.list` / `asset.search` rows already spell the containing FOLDER `packagePath`, while `asset.references` / `asset.dependencies` already spell this exact value `packageName`, so the other choice would have recreated a divergence one field over. The two per-verb audio workarounds are now redundant but harmless and were left alone. Tests (failure-direction, `Tests/Assets/TestAssetVerificationPathPreservation.cpp`): `PinWright.core.asset_save_honesty.add_asset_verification.HandlerAssetPathSurvives` sets `assetPath` from `GetPathName()` on a top-level /Game probe and requires it to survive (restoring the unconditional write makes it read the package path), plus the package-path, unset-field and `packageName` cases; `…add_asset_verification.ForeignAssetPathIsReplacedVisibly` requires a foreign path to be replaced AND the replacement to be visible. Both TUs compile clean (`-SingleFile -NoHotReloadFromIDE`, `Result: Succeeded`). **Not done, flagged for the tester:** the full suite was not run, so the ~50 verbs whose `assetPath` now reports the object path instead of the package path are unmeasured against their own tests; `TestUtils.h`'s shared `PackageFilenameFromAssetPath` trims at the dot and is unaffected, and the exact-value assertions found by grep are all on verbs outside the 50. Docs: the generalisable rule is `Docs/rpc-design.md` §21. Commits `f846bf87`, `c0852482`.
