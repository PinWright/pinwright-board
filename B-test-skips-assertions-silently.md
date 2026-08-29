---
id: B-test-skips-assertions-silently
title: "A test's substantive assertions skip themselves on a dark fixture and the test still reports green — 3 of 3 runs today measured nothing, and neither the suite log nor check_suite_log.py distinguishes that from a pass"
status: IN-REVIEW
severity: High
category: bug
tags: [testing, automation, render, capture_asset_preview, exposure-pin, false-green, conditional-skip, verdict-tooling, gpu-contention]
encounters: 1
lastSeen: 2026-08-18T13:15:00Z
---

# A green test that asserted almost nothing, and no way to tell from the outside

`PinWright.render.capture_asset_preview.PinnedCapturesAreIdentical`
(`Plugins/PinWright/Source/PinWright/Private/Tests/Render/TestCaptureExposurePin.cpp`, recalibrated
in `7600916d`) carries a lit-mode gate. Below `MinLitMeanLuminance = 0.03` the fixture is judged too
dark to measure, the test takes a `NOT MEASURED` path (`TestCaptureExposurePin.cpp:851`), asserts
only that the response's `blank` flag is derived from its own published statistics, and returns
`true`. Its **two substantive assertions never run**:

- same EV100 twice reproduces — `SamePinDifference < ToleranceMeanAbsDiff` (4.0), line 893;
- a different EV100 moves the image, so the pin is not inert — `CrossPinDifference >
  MinCrossEvMeanAbsDiff` (8.0), line 901.

The gate is correct in itself — asserting an exposure response against pixels that do not respond to
exposure would assert something the environment cannot show. The defect is that **taking the gate is
indistinguishable, from every consumer downstream, from passing.**

## Measured, this host, today

Three consecutive runs took the skip, all reading the same `meanLuminance 0.0078` black mode. Two
headless commandlet runs and one *visible* editor, so this is not the documented
interactive-vs-headless split:

| Log | UTC | Reading |
|---|---|---|
| `Saved/Logs/pw_suite_verify.log:33571` | `13:09:50` | `NOT MEASURED ... 0.0078 < 0.03 decoded` |
| `Saved/Logs/EAContentExamples58.log:2728` | `13:14:07` | same (visible editor) |
| `Saved/Logs/pw_render_interactive.log:2743` | `13:15:23` | same |

All three passed. The assertions **were** green earlier the same day —
`Saved/Logs/pw_suite_imgexp3.log:33784`, `11:21:15` UTC: `meanLuminance 0.0978`, same-pin
`meanAbsDiff 0.000`, cross-EV `23.127` over 8640/16384 px. (That run went red only against the
then-32.0 bar, which is what `7600916d` recalibrated to 8.0.)

The one environmental difference between the two windows: a **`DroneFootball` editor — an unrelated
project — started at 17:50 local (12:50 UTC) and holds the GPU.** Every lit reading predates it;
every black reading postdates it. So the fixture's lit mode is not reproducible while another
project's editor is running, and the whole test degrades to one flag-derivation check without
saying so.

## Why the silence is the bug

The skip is announced with `AddInfo`, which lands as an ordinary `LogAutomationController: Display:`
line with no `Warning:` and no machine-greppable marker. `Plugins/PinWright/Content/Python/check_suite_log.py`
(note: **not** `Content/Python/check_suite_log.py`) reconciles only `found` / `started` / `succeeded`
/ `failed` / `performed` — it has no outcome for "ran, asserted nothing". A suite whose green depends
on which other applications happen to be running is not a gate, and `7600916d`'s entire purpose was
to stop this test passing vacuously. **It now passes vacuously for a different reason.**

This is the project's signature defect class one level in: not a verb lying about a result, but a
*test* reporting a verdict about a property it never examined.

## Fix — proposed, in order

1. **Make the skip loud and machine-readable.** Emit at `Warning` with a stable marker (e.g.
   `PINWRIGHT_ASSERTIONS_SKIPPED: <TestName> <reason>`) so a skipped run is textually separable from
   a measured one, and so the reason travels with it.
2. **Teach the verdict tooling a third outcome.** `check_suite_log.py` should count and print
   `skipped` alongside started/succeeded/failed, and a run in which a *nominated* test skipped its
   assertions should not render as an unqualified green.
3. **Do not make the skip a hard failure by default.** Tempting, but wrong here: the commandlet
   cannot reach lit mode at all, so a hard fail trades a silent false-green for a permanent
   false-red on every headless host — the defect already filed and fixed as
   `B-tests-host-dependent-fixtures-hard-fail`. If a hard failure is wanted, gate it behind an
   explicit "this host is expected to render lit" opt-in, so the strictness is a stated
   environmental claim rather than an assumption.
4. **Investigate the mechanism.** Why does GPU contention from an unrelated editor change an
   `FAdvancedPreviewScene`'s lighting result at all? A preview scene losing its lighting because a
   second process holds the GPU is a rendering-path finding in its own right, and until it is
   understood the "interactive editor renders this lit" claim in the test's own comments is not
   dependable.

## Related — cross-reference, do not merge

- `B-exposure-pin-black-frame` — **distinct.** That one is a *verb* misreporting: `pinned: true` /
  `blank: false` on a degenerate all-black frame. This one is a *test* silently not measuring. Same
  fixture and same black frame, opposite ends of the reporting chain; they share a root-cause
  candidate (item 4 above) but neither fix implies the other.
- `B-suite-log-completeness-unverifiable` — adjacent: a killed suite greps identically to a drained
  one. That is missing *tests*; this is a present test missing its *assertions*. Both land on the
  same consumer, so remedy 2 here should be implemented with that ticket's verdict rules, not beside
  them.
- `B-tests-host-dependent-fixtures-hard-fail` — the precedent that makes remedy 3 a "no".
- `B-unlit-level-capture-no-warning` — same shape one layer out: a dark result nobody warns about.

## History
- `#1-three-of-three-runs-measured-nothing` `OPEN` reporter — `PinWright.render.capture_asset_preview.PinnedCapturesAreIdentical` reports green while both of its substantive assertions are skipped, and nothing downstream can tell. The lit-mode gate `MinLitMeanLuminance = 0.03` (`TestCaptureExposurePin.cpp:851`) routes a dark fixture to a `NOT MEASURED` path that asserts only the `blank`-flag derivation and returns success; the tolerance check (`SamePinDifference < 4.0`, line 893) and the non-inert check (`CrossPinDifference > 8.0`, line 901) never execute. Measured on this host: three consecutive runs — `Saved/Logs/pw_suite_verify.log:33571` at 13:09:50 UTC, `Saved/Logs/EAContentExamples58.log:2728` at 13:14:07 UTC in a **visible** editor, and `Saved/Logs/pw_render_interactive.log:2743` at 13:15:23 UTC — all read `meanLuminance 0.0078` and all passed, so this is not the interactive-vs-headless split the test's own comments describe. The same assertions were genuinely green earlier the same day at `Saved/Logs/pw_suite_imgexp3.log:33784`, 11:21:15 UTC: `meanLuminance 0.0978`, same-pin `meanAbsDiff 0.000`, cross-EV `23.127` over 8640/16384 px. The only environmental difference is a `DroneFootball` editor from an unrelated project, started 12:50 UTC, that holds the GPU — every lit reading predates it, every black reading postdates it. The skip is emitted via `AddInfo` as a plain `LogAutomationController: Display:` line with no `Warning` and no marker, and `Plugins/PinWright/Content/Python/check_suite_log.py` reconciles only found/started/succeeded/failed/performed, so it has no outcome for "ran but asserted nothing". Remedies in the body, in order: a loud machine-readable skip marker; a third `skipped` outcome in the verdict tooling; **not** a default hard failure (that recreates `B-tests-host-dependent-fixtures-hard-fail` on every headless host — gate it behind an explicit expected-lit opt-in instead); and an investigation into why an unrelated editor's GPU load changes an `FAdvancedPreviewScene`'s lighting at all. Severity High per the rubric's silent-false-success class, not Critical — no editor crash and no data-corrupting write — and the reach modifier cannot lift it further, since the rubric defines Critical by impact class alone.
- `#2-additional-marker-is-only-partial` `OPEN` reporter — Additional evidence: **Adversarial review A — the first remedy landed, but the false-green remains.** Actuality: **PARTIAL**. Framing: title and High severity remain accurate; current `EAContentExamples58` source emits a stable `AddWarning` marker at `<X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWright\Private\Tests\Render\TestCaptureExposurePin.cpp:140-146,1281-1295>`, but consumers still treat it as pass evidence. Proposed fix: **INCOMPLETE**, because `<X:\src\unreal\EAContentExamples58\Plugins\PinWright\Content\Python\check_suite_log.py:26-33,107-112>` has only three states and never parses the marker; `<X:\src\unreal\EAContentExamples58\Plugins\PinWright\Content\Python\mcp_proxy.py:981-985,1042-1058>` likewise has no skipped state and accepts `succeededWithWarnings`, while CI counts those warnings as passed at `<X:\src\unreal\EAContentExamples58\Plugins\PinWright\.github\workflows\ci.yml:201-215>`. Evidence: the live checker classified `<X:\src\unreal\EAContentExamples58\Saved\Logs\pw_fix_check.log:3080,3088>` (two `PINWRIGHT_ASSERTIONS_SKIPPED` warnings; 131/131 tests completed) as `COMPLETED_CLEAN`; the existing unit test explicitly preserves warning-as-success at `<X:\src\unreal\EAContentExamples58\Plugins\PinWright\Content\Python\tests\test_mcp_proxy_editor_start.py:1489-1496>`, with no marker case. Runtime: **NOT VERIFIED** (no Unreal launch in this review); static runner behavior is verified. Recommendation: **KEEP**; finish one shared marker-aware parser/classifier and CI/report gate, add focused marker tests, and keep ordinary warnings non-fatal; separately investigate the GPU/light-fixture cause.
- `#3-additional-marker-does-not-fix-verdict` `OPEN` reporter — Additional evidence: **Adversarial review B — A's partial-defect finding survives, but its framing and policy need correction.** Actuality: **PARTIAL**. Framing: the old claim that neither the suite log nor downstream evidence distinguishes a skip is stale: current source emits `LogAutomationController: Warning: PINWRIGHT_ASSERTIONS_SKIPPED` at `<X:\src\unreal\EAContentExamples58\Saved\Logs\pw_fix_check.log:3080,3088>`, and the test ID is now `PinnedCapturesReproduceWithinTolerance` at `<X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWright\Private\Tests\Render\TestCaptureExposurePin.cpp:1027-1029>`. The remaining automated false-green is confirmed: `<X:\src\unreal\EAContentExamples58\Plugins\PinWright\Content\Python\check_suite_log.py:32-37,107-113>` ignores the marker, `<X:\src\unreal\EAContentExamples58\Plugins\PinWright\Content\Python\mcp_proxy.py:816-901,981-1011>` parses no skip state, and CI treats `succeededWithWarnings` as passed at `<X:\src\unreal\EAContentExamples58\Plugins\PinWright\.github\workflows\ci.yml:201-215>`. High remains justified for this machine/CI false-success. Proposed fix: **INCOMPLETE**, because a shared parser must recognize this stable marker specifically (not all warnings), preserve test/reason, and define explicit strictness such as `--expected-lit`/`--strict-skips`; default headless skips must remain non-fatal, while an opted-in gate must not pass. Runtime: **NOT VERIFIED** (no Unreal launch); 175 current Python tests pass, and the live checker still prints `COMPLETED_CLEAN` for the marker-bearing log. Recommendation: **REFRAME**; narrow the title/body to the machine-verdict false-green after marker landing, then add marker and ordinary-warning fixtures across checker, proxy, and CI.
- `#4-verdict-gate-and-one-literal` `IN-REVIEW` developer — Fixed across three commits in the plugin repo, and the ticket itself was stale in three ways worth recording. **What landed.** `e003e7b8` gave the verdict tooling the third outcome remedy 2 asked for: `COMPLETED_WITH_SKIPS`, reconciled from the marker by one shared ladder that both `check_suite_log` and `editor_run_tests` read, so the two entry points cannot drift; it maps to `EDITOR_TESTS_SKIPPED`, exits nonzero, and is deliberately NOT a red test — remedy 3 held, so a headless host still passes rather than re-creating `B-tests-host-dependent-fixtures-hard-fail`. The count is read off the LOG even when a valid automation report wins the report-first evidence selection, because `index.json` has no skip field and reading one off a report returns a structural zero. `f9f3a7b7` fixed the half nobody had noticed: twenty markers in `Tests/Render/` were emitted through `AddInfo`, which lands as `LogAutomationController: Display:` and is never written to the automation log at all — so they were invisible rather than merely quiet, and no log-based checker could ever have counted them. Measured on a 4242-test log carrying exactly 8484 = 4242 x 2 `LogAutomationController: Display:` lines (Started and Completed only) and zero marker hits. `28896975` closed the last structural hole: the literal was written out at 32 sites — nine file-local helpers under five different names plus twenty-three inline `AddWarning` calls, each carrying a comment telling the next reader to change it "in both places or in neither" — and it was ALREADY spelled two ways, `TestAssetPreviewSubjects.cpp` and `TestAssetPreviewSubjectTime.cpp` omitting the trailing colon. Counts came out right only because `mcp_proxy.py` spells the colon `:?`; a future cleanup tightening that regex would have dropped those two files out of the gate silently. There is now one literal and one emitter, `PinWrightTestSkip::SkipAssertions` in `Source/PinWright/Private/Tests/TestSkipReporting.h`, so the property is structural rather than aspirational, and three sites that printed no test id now carry one. `a0330754` pins the parser's remaining tolerance on purpose (7 tests; the Python suite is 225, was 218) and `28896975` adds `PinWright.infra.skip_marker.*`, which asserts the emitter puts the exact wire text on the WARNING channel — emitted into a scoped `FAutomationTestBase` probe rather than into the running test, because emitting a real marker would make `check_suite_log` refuse `COMPLETED_CLEAN` on every run forever. That test exists because the mechanism is otherwise unprovable from a green run: the last full suite was 4288/4288/0 with ZERO markers, so no skip site fired and the `AddInfo`->`AddWarning` conversion had never been exercised end to end. **Where this ticket is stale.** (a) The test id is now `PinnedCapturesReproduceWithinTolerance` (`TestCaptureExposurePin.cpp:989`), not `PinnedCapturesAreIdentical`; entry `#3` already caught this. (b) Every line number in the body has moved — the lit gate is at `:1242`, not `:851`; the tolerance assertion at `:1317`, not `:893`; the non-inert assertion at `:1325`, not `:901`. Cite the constants (`MinLitMeanLuminance`, `ToleranceMeanAbsDiff`, `MinCrossEvMeanAbsDiff`) rather than the lines. (c) The body's strongest implicit claim — that the recalibrated 4.0 / 8.0 bars had never been observed passing, since the one green run predated `7600916d` and went red against the then-32.0 bar — is now FALSE. `Saved/Logs/pw_final_suite.log:38181`, a **headless `-RenderOffscreen` commandlet** run, records `ev100 -1.0: meanLuminance 0.4797. same-pin meanAbsDiff 0.412 over 9639/16384 px; ev100 -1.0 vs 5.0 meanAbsDiff 117.154 over 16276/16384 px. tolerance 4.0.` with `Result={Success}` at `:38179`. Both bars pass with two orders of magnitude of headroom, at a mean luminance 16x the 0.03 lit gate — so the fixture reaches lit mode headlessly, and the "commandlet cannot reach lit mode at all" premise behind remedy 3 does not hold either (remedy 3 is still right, for the different reason that a hard fail would punish any host that legitimately cannot render). Remedy 4 — why an unrelated project's editor holding the GPU changes an `FAdvancedPreviewScene`'s lighting — remains **uninvestigated** and is the reason this is IN-REVIEW rather than a claim of completeness. A tester should verify against a run taken while another project's editor holds the GPU: the marker must appear, `check_suite_log` must return `COMPLETED_WITH_SKIPS` and exit nonzero, and the named test must be attributed in `skippedTests`.
- `#5-adoption-residue-split-out` `IN-REVIEW` reporter — "Scope note for whoever tests this ticket: verifying the named test and the verdict gate does NOT close the defect class. A static sweep of every PinWright*/Private/Tests tree on 2026-08-29 found 357-393 conditional skips (137 files; PinWright 331, PinWrightGeometry 62) still written as a bare AddWarning followed by `return true;`, emitting nothing check_suite_log.py can count. That residue is tracked separately as B-tests-warn-and-pass-without-skip-marker, including the static-guard exit criterion; this ticket stays about the mechanism and the named test."
