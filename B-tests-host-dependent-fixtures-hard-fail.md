---
id: B-tests-host-dependent-fixtures-hard-fail
title: "Host-dependent fixture tests hard-fail on hosts without Lyra mannequin content / with CommonGame"
status: DONE
severity: High
category: bug
tags: [testing, fixtures, host-compat, agir, crir]
---

# Host-dependent fixture tests hard-fail on hosts without Lyra mannequin content / with CommonGame

29 automation tests hard-fail in the PDS host project (a Lyra fork) because
they assume host-specific content. 28 of them load Lyra mannequin content by
absolute `/Game/` path — `/Game/Characters/Mannequins/Animations/ABP_Manny`
(AnimBP and its Skeleton) and `/Game/Characters/Mannequins/Rigs/CR_Mannequin_Body`
(Control Rig) — which PDS deleted, so `LoadObject` returns null and the
`TestNotNull("Mannequin AnimBP fixture loaded")`-style asserts fail. The 29th
(`PinWright.ui.activatable.LayerTagHostUnavailableWithoutCommonGame`) asserts
that CommonGame's `UPrimaryGameLayout` is ABSENT, which is false on any
Lyra-fork host that ships CommonGame. All of these tests are valid in hosts
that have (or lack) the content; they must not hard-fail in hosts that don't
match their premise. Affected files: Tests/Assets/TestAnimGraphHandlers.cpp
(18 tests), TestAGIRBlendSpace / TestAGIRCachedPose / TestAGIRClassifier /
TestAGIRCustomTransition / TestAGIRLayeredBlend / TestAGIRLinkedInputPose /
TestAGIRStateMachineStateBody.cpp (1 each), TestCreateBlendSpaceAxisConfig.cpp,
TestCRIRRoundTrip.cpp (HierarchyEmit), Tests/Utility/TestAssetDumpAnimGraph.cpp,
Tests/UI/TestUiActivatableLayerTag.cpp.

**Fix:** host-absence = skip, load-failure = fail. Added
`PINWRIGHT_SKIP_IF_FIXTURE_MISSING(PackagePath)` to `Tests/TestUtils.h`: it runs
`FPackageName::DoesPackageExist` on the fixture's package and, when absent,
emits an `AddInfo` note prefixed with the audit-greppable `FIXTURE-SKIP:` token
and passes the test early. Every mannequin-loading test gates with it before
`LoadObject`; a package that exists but fails to load still hard-fails.
`TestAssetDumpAnimGraph.cpp` instead branches on the dump handler's
`ASSET_FILE_MISSING` error code (its pre-LoadObject existence check) for the
skip, keeping all other codes fatal. The CommonGame test is invert-guarded: if
`/Script/CommonGame.PrimaryGameLayout` resolves on this host, it emits the
FIXTURE-SKIP note and passes. Convention documented in
`docs/test-organization.md` ("Host-Dependent Fixtures").

## History
- `#1-29-hard-failures-on-pds` `OPEN` reporter — 29 tests hard-fail in the PDS host: 28 load Lyra mannequin content (ABP_Manny AnimBP/Skeleton, CR_Mannequin_Body Control Rig) that PDS deleted; 1 asserts CommonGame is absent, false on a Lyra fork.
- `#2-fixture-skip-gate-implemented` `IN-REVIEW` developer — "Added PINWRIGHT_SKIP_IF_FIXTURE_MISSING (DoesPackageExist gate + FIXTURE-SKIP AddInfo + early pass) to Tests/TestUtils.h and gated all 28 mannequin-loading tests; converted TestAssetDumpAnimGraph's ASSET_FILE_MISSING branch to a FIXTURE-SKIP pass (other codes stay fatal); invert-guarded LayerTagHostUnavailableWithoutCommonGame to skip when UPrimaryGameLayout resolves; documented the convention in docs/test-organization.md. Awaiting a verification run of the suite in the PDS host."
- `#3-full-suite-verify-pass` `DONE` tester — "Full PinWright suite in the PDS host (UnrealEditor-Cmd, -unattended -nocefaccelpaint, Saved/Logs/PinWrightTestsFull2.log): 3570 started, 3570 passed, 0 failed, exit code 0, exactly 29 FIXTURE-SKIP notes emitted (28 mannequin-content gates + 1 CommonGame invert-guard). Baseline before fix was 3541/3570 with the same 29 as hard failures."
