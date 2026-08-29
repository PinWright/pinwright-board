---
id: B-tests-warn-and-pass-without-skip-marker
title: "393 conditional skips across 137 test files warn-and-return-true without the PINWRIGHT_ASSERTIONS_SKIPPED marker, so a host that measured nothing still classifies COMPLETED_CLEAN"
status: OPEN
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
