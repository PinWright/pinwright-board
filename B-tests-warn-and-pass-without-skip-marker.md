---
id: B-tests-warn-and-pass-without-skip-marker
title: "393 conditional skips across 137 test files warn-and-return-true without the PINWRIGHT_ASSERTIONS_SKIPPED marker, so a host that measured nothing still classifies COMPLETED_CLEAN"
status: IN-REVIEW
severity: High
category: bug
tags: [testing, automation, false-green, conditional-skip, skip-marker, verdict-tooling, check-suite-log]
encounters: 1
lastSeen: 2026-08-29T09:00:00Z
---

# The marker mechanism is right; most of the emitters never got wired to it

`PinWrightTestSkip::SkipAssertions` (`Source/PinWright/Private/Tests/TestSkipReporting.h`) is the
house mechanism for a test that steps over its substantive assertions: it emits
`PINWRIGHT_ASSERTIONS_SKIPPED:` through `AddWarning`, `Content/Python/check_suite_log.py` counts it,
and the run classifies `COMPLETED_WITH_SKIPS` instead of `COMPLETED_CLEAN`. That mechanism works and
has ~180 call sites.

Everywhere else the same early return is still written as a bare
`AddWarning(TEXT("... skipping ..."))` followed by `return true;`. In the suite log that is
**byte-indistinguishable from a genuine pass**: the test lands `Result={Success}`, `started ==
succeeded`, the drain marker is present, `skipped=0`, exit code 0. This is the parent defect
`B-test-skips-assertions-silently` reproduced at scale — one named test was fixed, the class was not.

## Census, measured 2026-08-29 over `Plugins/PinWright/Source/PinWright*/Private/Tests/`

Two independent static shapes were run over every test `.cpp`/`.h`, excluding sites already routed
through `PinWrightTestSkip`:

- **Shape A** — `AddWarning` reached by a `return true;` within 8 lines with no intervening
  assertion or `AddError`: **365** sites.
- **Shape B** — an `AddWarning` whose (possibly multi-line) message declares a skip
  (`skip` / `cannot` / `could not` / `unavailable` / `not available`): **385** sites.
- **Intersection (high confidence): 357.** **Union (outer bound): 393, across 137 files.**

By module, on the union:

| module | sites |
| --- | --- |
| `PinWright` | 331 |
| `PinWrightGeometry` | 62 |
| `PinWrightPCG` | 0 (converted, see History `#1`) |
| `PinWrightChooser` / `PinWrightCommonUI` / `PinWrightPoseSearch` / `PinWrightRecorder` | 0 (no such sites) |

Densest files: `Tests/World/TestEnvironmentHandlers.cpp` 20, `Tests/Gameplay/TestGASHandlers.cpp` 15,
`Tests/Actor/TestActorLabelResolution.cpp` 12, `Tests/World/TestVolumeHandlers.cpp` 10,
`Tests/Environment/TestLandscapePaintLayerHonesty.cpp` 10,
`Tests/Actor/TestActorErgonomicsHandlers.cpp` 10,
`Tests/Actor/TestActorSpawnMaterialAssignment.cpp` 9,
`PinWrightGeometry/.../TestGeometrySkeletalMeshRoundTrip.cpp` 8.

**Both shapes have false positives** — a test that asserted, then warned, then returned true reads
the same statically. Both also have false negatives: two real skips in
`PinWrightPCG/.../TestPCGGenerateHandler.cpp` (the inline-resolution and inline-readback paths of
`TicketedKickoffDoesNotBlock`) run one residual assertion before abandoning the ticket contract, and
Shape A's lookahead therefore stepped past them. Treat 357–393 as a range, and judge each site when
converting it.

## Why the totals in `Plugins/PinWright/CLAUDE.md` do not settle this

The 4576-test measurement of 2026-08-28 reported `skipped=4`. Those four are the sites that already
carry the marker. The 393 sites above emit nothing countable, so any of them that fired on that host
is inside `succeeded=4576` with no trace. Every "0 skips" green baseline recorded in that file is a
statement about the marked population only.

## Fix

Convert each site to `PinWrightTestSkip::SkipAssertions(*this, TEXT("<reason-slug>"), <detail>)` and
add `#include "Tests/TestSkipReporting.h"` (sub-modules resolve that path through their
`PrivateIncludePaths` into `Source/PinWright/Private`). Reuse the established slug vocabulary
(`no-editor-world` 21 + `no_editor_world` 18, `fixture-unavailable` 12, `cvar-pinned` 10,
`capture-unavailable` 9, `cvar-absent` 5, …) rather than minting near-duplicates; the detail string
carries the measurement that produced the decision.

**Judge, do not mass-rewrite.** Three classes turned up in the first batch and will recur:

1. **Unreachable guards.** `if (!NewObject<T>(...))` never fires — `NewObject` aborts rather than
   returning null. Five such branches were found in `TestPCGGenerateHandler.cpp` alone. They are
   dead code wearing a skip's clothes; converting them is harmless but deleting them is better.
2. **Near-dead host guards.** `no-editor-world` on an `EAutomationTestFlags::EditorContext` test is
   true on every host that can run the suite. 39 marked call sites already do this, so it is the
   house pattern and not worth churning — but it inflates any count of "real" skips.
3. **Genuine, and currently invisible.** The two `TicketedKickoffDoesNotBlock` paths above fire on
   any bare automation world with no live PCG scheduler. These are the sites the marker exists for.

**Exit criterion — the static guard.** Once the residue is zero, add a stdlib-only
`Content/Python/check_test_skips.py` in the shape of `check_test_ids.py` (own CI job, self-tested
under `Content/Python/tests/`) asserting that no test source contains a bare warn-and-pass. It is
deliberately **not** added now: with 393 live violations it would be red from birth, and a
born-red gate gets an allowlist or gets switched off. The runtime counterpart already exists and
needs nothing — `PinWright.infra.skip_marker.*`
(`Tests/Infra/TestSkipMarkerEmission.cpp`) asserts the emitter's wire text and countability, which
is the emitter's contract, not the call sites' adoption.

## History
- `#1-census-and-pcg-conversion` `OPEN` reporter — "Swept all seven module test trees for early returns that warn and pass. Census: 365 (Shape A) / 385 (Shape B) / 357 intersection / 393 union across 137 files; PinWright 331, PinWrightGeometry 62, others 0. Converted 23 sites in this pass and left the rest as this ticket's residue: all 19 in PinWrightPCG (TestPCGGenerateHandler.cpp 16 — including the two the static shape missed, TestPCGGraphHandlers.cpp 3), and the 4 remaining bare guards in PinWrightGeometry/.../TestMeshMeasureHandler.cpp, whose other 3 tests already used the marker and left the file mixed. Reused existing slugs (no-editor-world, fixture-unavailable) and minted three where nothing fitted: fixture-did-not-populate, generation-resolved-inline, inline-readback-not-ticketed. Flagged but did not delete five unreachable `if (!NewObject<T>(...))` guards in TestPCGGenerateHandler.cpp. NOT COMPILED and NOT RUN — a full automation suite held the DLLs for the duration of this pass."
- `#2-sweep-scanner-and-ci-gate` `IN-REVIEW` developer — "Residue is zero and the gate is in place. Wrote `Content/Python/check_test_skips.py`, a stdlib-only source scan flagging any `AddWarning(...)` whose next statement is `return true;` (preprocessor lines stepped over; comment-stripped; test sources only), self-tested by `Content/Python/tests/test_warn_and_pass_scan.py` (17 cases incl. the defect flagged, a converted SkipAssertions site NOT flagged, warn-then-assert NOT flagged, `return false` NOT flagged, an empty tree failing closed, and the real tree asserted clean), and wired as its own CI job `test-skip-scan` in `.github/workflows/ci.yml`, self-test first, exactly like `test-id-scan`. Its own census read 328 sites in 105 files, not the ticket's 393/137: the earlier shapes counted a `)` inside a message literal as the call's close and did not exempt sites already spelling the marker inline. All 328 converted (14 parallel agents, one file-disjoint batch each), plus 19 the scan CANNOT see, found and converted by hand: 10 helper-frame emitters where a helper taking `FAutomationTestBase&` warns and returns false/nullptr and every caller turns that into `return true` (`FParseFixture::Init` alone fronts 8 tests; also 5 sequencer probe-sequence helpers, `TestPPVSetterEchoesAppliedValues` in `Tests/TestUtils.h`, `BuildActiveWindowOrSkip`, `BuildOrSkip`, `CaptureOrSkip`), 2 else-branch sites sharing an outer return (`lighting.set_exposure`, whose override-bit assertions ARE that test's whole coverage), 4 with a cleanup statement between warn and return, 2 separated only by `#endif` (the scan was then widened to that shape, with a self-test), and 1 inline marker literal re-routed through the house emitter. Call sites 256 -> 634; 174 files now carry `Tests/TestSkipReporting.h`; slug vocabulary 92 -> 128 distinct, reusing `no-editor-world` (216) and `fixture-unavailable` (56) wherever they fitted and minting ~33 where nothing did (`test-port-unbindable`, `engine-version-unsupported`, `sequence-factory-unavailable`, `optional-plugin-not-shipped`, ...). No assertion weakened, no guard deleted, no test logic changed, no `PINWRIGHT_WARNING_IS_NOT_A_SKIP` waiver used anywhere. VERDICT TOOLING NEEDED NO CHANGE and this was verified rather than assumed: a synthetic drained 3-test log with zero failures and one marker classifies `COMPLETED_WITH_SKIPS` with `skipped=1`, names the test, and `check_suite_log.py` exits 1; `tests/test_skip_marker_literal.py` (7) and the skip cases of `tests/test_suite_verdict_gates.py` all pass. EXPECT THE MEASURED SKIP COUNT TO STAY NEAR ITS CURRENT VALUE, not to jump by 347 - most of these guards are near-dead on any host that can run the suite (216 are `no-editor-world` on `EditorContext` tests, which is false wherever the suite runs at all), so a small delta is not evidence the conversion failed and a large one is not a regression. NOT COMPILED and NOT RUN per the wave's instruction; a 634-site mechanical sweep needs a compile before this closes, and every call site was structurally validated instead (balanced parens, 3 top-level args, lowercase TEXT slug, terminating semicolon). Python suite 191 tests, 188 pass - the 3 reds are crash/fatal-banner cases in `tests/test_suite_verdict_gates.py` broken by another agent's concurrent in-flight edit to `mcp_proxy.py` (modified 23s before the run), not by anything here; `mcp_proxy.py` and `check_suite_log.py` were not touched. LARGEST REMAINING FALSE-GREEN POPULATION, deliberately NOT swept here and needing its own ticket: `AddInfo(...); return true;` - 227 sites in 70 files, plus the `PINWRIGHT_SKIP_IF_FIXTURE_MISSING` / `PINWRIGHT_SKIP_IF_ALL_FIXTURES_MISSING` macros in `Tests/TestUtils.h` and their 50 call sites. `AddInfo` events never reach the automation log at all, so those skips are worse than uncounted - they are unreadable, and `FIXTURE-SKIP:` being 'the audit token' in `Docs/test-organization.md` is a grep over a log line that is never written. Widening the scan to `AddInfo` is the right follow-up but must come AFTER that sweep: a gate born red gets an allowlist or gets switched off, which is this ticket's own stated exit criterion. Docs: new 'Conditional Skips Must Be Countable' section in `Docs/test-organization.md` carrying the emitter, both enforcing checks, the at-the-site opt-out, and that gap."
- `#3-addinfo-sibling-filed-and-two-scanner-claims-corrected` `IN-REVIEW` reporter — **Status deliberately NOT changed; no code touched.** The `AddInfo` hole `#2`'s scanner names as "the largest remaining hole" is now **`B-tests-addinfo-skip-not-marked`** (OPEN, High — level with this ticket, not above it). Two load-bearing claims in `Content/Python/check_test_skips.py:49-53` are **wrong** and should be corrected in whichever commit closes that ticket, because a source comment asserting the audit token is never written will send the next reader after a log-plumbing bug that does not exist. **(1) The census does not reproduce.** The comment says "227 sites in 70 files". Re-derived over all eight modules with `AddInfo\([^;]*\);\s*\n\s*return true;`: **128 occurrences in 43 files**. Loosening to allow one intervening statement, restricted to `PinWright/Private/Tests`, reaches only 150 in 47. Total `AddInfo(` population is 423 across 114 files, all inside `*/Tests/`. The macro figure is fine: `PINWRIGHT_SKIP_IF_FIXTURE_MISSING` 43 + `PINWRIGHT_SKIP_IF_ALL_FIXTURES_MISSING` 4 = **47 across 18 files**, against the stated "50". This is the second census on this defect class to come in materially wrong — `#1` filed 393 here and `#2` had to re-derive — so re-derive before acting on any number. **(2) "`AddInfo` events never reach the automation log at all" is false.** `FAutomationControllerManager::ReportAutomationResult` walks `Results.GetEntries()` and logs `EAutomationEventType::Info` as `UE_LOGF(LogAutomationController, Log, ...)` (`C:\UE_5.8\Engine\Source\Developer\AutomationController\Private\AutomationControllerManager.cpp:1685-1693`), and a retained run log proves it: `Saved/Logs/Automation_PinWright_verify2.log` carries **36** `LogAutomationController: FIXTURE-SKIP: ...` lines. So `FIXTURE-SKIP:` works exactly as `Docs/test-organization.md:260` documents. The accurate mechanism is the one `TestSkipReporting.h:38-41` already states — the entry is logged and greppable by its own token but carries no severity and no marker, so `check_suite_log.py` (which counts `PINWRIGHT_ASSERTIONS_SKIPPED` only) cannot see it and the run still classifies `COMPLETED_CLEAN`. Nothing here contradicts this ticket's own fix or `#2`'s gate; the gate's `AddWarning`-only scope (`check_test_skips.py:99`, `:291`) is confirmed correct and its "widen after the sweep, not before" note is endorsed. **Scope for this ticket's tester is unchanged and is the `AddWarning` shape only.**
