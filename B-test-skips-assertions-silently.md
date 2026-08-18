---
id: B-test-skips-assertions-silently
title: "A test's substantive assertions skip themselves on a dark fixture and the test still reports green — 3 of 3 runs today measured nothing, and neither the suite log nor check_suite_log.py distinguishes that from a pass"
status: OPEN
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
