---
id: F-interleaved-ab-measurement
title: "Nothing in PinWright takes a before/after measurement that cancels machine drift, and drift on this host was large enough to invent three conclusions: two identical baseline sweeps 50 minutes apart came out 18-22% faster with nothing changed (F6 24.27 -> 19.81 ms, W2 13.63 -> 10.60 ms), so -27%, -16% and -20% were all artefacts and the last re-measured interleaved at -3.3% — p10 defends against a stall inside a capture, not against the baseline moving between two of them"
status: OPEN
severity: Medium
category: feature
tags: [performance, measurement, benchmark, ab-testing, interleaved, drift, baseline, percentiles, methodology, profiling, reproducibility, frame-time]
encounters: 1
lastSeen: 2026-08-30T17:55:00+03:00
---

# Two measurements taken an hour apart are not comparable, and nothing says so

A performance pass on `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8, editor
build 13:32 / plugin `d8f1bc32`) ran the same six-pose baseline sweep twice, once before a tuning
pass and once after it, with **nothing changed between them**:

| pose | base1 (ms) | base2 (ms) | drift |
|---|---|---|---|
| F6 deep | 24.27 | 19.81 | -18% |
| F1 canopy | 19.88 | 16.31 | -18% |
| W1 wide | 16.11 | 13.27 | -18% |
| W2 meadow | 13.63 | 10.60 | **-22%** |

Source: `Docs/map/vegetation-performance.md:378-399` on that host, table at `:383-388`.

**Every A/B taken against `base1` was therefore wrong, and the errors were large enough to invent
conclusions.** `r.Shadow.Virtual.ResolutionLodBiasDirectional 0` read as **-27%**,
`r.Nanite.MaxPixelsPerEdge 2` as **-16%**, a per-species WPO-disable-distance table as **-20%**
(`:390-394`). Re-measured interleaved, the per-species table is **-3.3%** — the effect was 6x
smaller than the artefact that hid it. *Stated precisely because it is easy to overclaim:* only the
per-species table was re-measured against its own non-interleaved figure. `MaxPixelsPerEdge 2`
appears in the interleaved lever table at `:337-346` at **-5.8%** at F6, which is its honest
counterpart, and `ResolutionLodBiasDirectional 0` does not appear there at all.

**Percentiles do not help.** The same document's hazard 4 (`:168`) establishes p10 as the right
statistic *within* a capture, because a stall can only add time — p10 reproduced to 0.07 ms across
independent samples where the median wandered by 4 ms. Hazard 8 says why that is not enough
(`:392-394`): p10 defends against a stall inside a capture, not against the baseline moving between
two captures. A perfectly-measured A and a perfectly-measured B taken an hour apart still differ by
whatever the machine did in between.

## Nothing in the plugin measures this way, or any way

Swept `Plugins/PinWright/Source/` (1923 `.cpp`/`.h`) for `paired`, `interleav`, `ab_test`,
`abTest`, `benchmark`, `baseline`, `repeats`, `alternat`:

- `paired` / `interleav` — hits are all comment prose about unrelated things (paired arrays, paired
  `bOverride` flags, deinterleaved PCM in `AudioGen/PwAudioDecode.cpp`, interleaved trace threads).
  **Zero measurement semantics.**
- `benchmark` — one verb, `performance.run_benchmark`
  (`Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp:860`, body `:860-907`). It takes
  a `duration` and an ignored `type`, issues `stat startfile`, waits on an `FTSTicker` and
  finalizes. **It has no A side and no B side, no repeats, and no statistics** — and on UE 5.8 the
  capture is compiled out (`PINWRIGHT_HAS_STATS_FILE_CAPTURE`, `:27-31`), so its result is
  `{captured: false}` (`:894-900`, comment `:891-893`).
- `baseline` — `performance.apply_baseline_settings` is a curated CVar bundle, not a measurement
  baseline.
- `repeats` / `alternat` — no measurement hits.

No frame-time readback exists to build on either; that gap is
`F-performance-frame-time-statistics` (OPEN, Medium), and this ticket sits on top of it: a
measurement *method* needs a measurement *primitive*.

## The working implementation lives outside the plugin, on one map

`X:/src/unreal/EAContentExamples58/dev/perf/pw_paired_ab.py` (3996 bytes, tracked in the **map**
repo, not the plugin — `Plugins/PinWright/` has no `dev/` directory). Its own header states the
principle: *"Alternates config A and config B `repeats` times inside ONE tick-driven run, so slow
machine-level drift cancels instead of being attributed to the change under test."*

Mechanically: one `unreal.register_slate_post_tick_callback`, timing from `time.perf_counter()`
deltas per tick; at each config switch it walks every `InstancedStaticMeshComponent`, matches by
static-mesh name, applies that config's properties and executes its CVar list; per step it sets the
camera via `UnrealEditorSubsystem.set_level_viewport_camera_info`, burns `warmFrames` (default 90)
and samples `dwellFrames` (default 90); reports `{config, pose, frames, p10, p50}` per step to JSON
and re-applies config A at the end. Its A repeats hold to +/-0.1 ms (19.91 / 20.00 / 19.90 at F6,
`vegetation-performance.md:397`).

**One detail worth carrying into any implementation rather than assuming:** its plan is
`for _ in range(repeats): for cfg in ("A","B"): for pose in poses` (`pw_paired_ab.py:93-97`), i.e.
A-all-poses then B-all-poses, repeated — **block alternation, not strict per-pose ABAB**. That
cancels drift on the minutes timescale hazard 8 measured; it would not cancel a confounder
switching on the same few-second cadence. "Interleaved" invites a reader to assume strict ABAB, and
it is not.

The older, non-interleaved driver is still beside it (`dev/perf/run_ab.sh`) — that is the shape
that produced the -27 / -16 / -20% figures.

**`Docs/map/vegetation-performance.md:399` says in as many words where the general form belongs:**
*"The universal form of the lesson belongs in PinWright, not here."* Nothing on the board owns it.

## Ask

Either of two shapes; the first is the real ask and the second is the cheap floor.

1. **An interleaved measurement verb.** Given a list of poses and two named configurations
   (component property sets and/or CVar sets), alternate them inside one tick-driven run for N
   repeats, and return per-configuration statistics plus the **A-repeat spread** — that last number
   is what tells the caller whether the run was stable enough to believe at all, and is why the
   `+/-0.1 ms` figure above is the harness's own acceptance criterion. It must re-apply the entry
   configuration on every exit path, including error paths, because a half-applied A/B leaves the
   level in a state nobody asked for.

   Two properties this must not omit: the **A-repeat spread** as a first-class field (a run whose A
   side wanders by 2 ms cannot report a 1 ms delta), and a **same-scene guard** — `primitivesDrawn`
   or draw calls per step, since in a shared editor another agent changing the level mid-run is not
   hypothetical (measured: 95,648 -> 151,568 mid-comparison,
   `vegetation-performance.md:174-176`).

2. **Failing a verb, a documented recipe** in the plugin's own wiki — under `performance` or
   `insights` — stating that a non-interleaved before/after in a live editor measures drift, giving
   the measured magnitude, and describing the alternation. That is strictly worse (it is advice a
   caller has to remember and hand-implement) but it costs one page and closes the "nobody was
   told" half.

## Distinct from

- **`F-performance-frame-time-statistics`** (OPEN, Medium) — the primitive this method needs: no
  `performance.*` verb returns a measured quantity at all. **Neither closes the other**: a
  frame-time verb still lets a caller take A now and B in an hour and believe the difference, which
  is exactly what happened here; and an interleaving harness with nothing to sample is inert.
  Sequencing: the primitive lands first.
- **`E-viewport-info-no-render-resolution`** (OPEN, Medium) — one *named* confounder (the editor
  window restoring at a different size; measured 1515x939, 738x502, 1024x726, the small one reading
  ~50% faster). Interleaving cancels every slow confounder including the unnamed ones, which is the
  difference between reporting a denominator and controlling for one. Complementary.
- **`F-report-concurrent-engine-processes`** (OPEN, Medium) — the sibling filed from this same
  session for a confounder that is *fast* and *categorical* rather than slow and gradual, and which
  interleaving therefore does not cancel.
- **`B-performance-run-benchmark-no-completion-signal`** (DONE, Medium) — fixed the async
  completion signal on `run_benchmark`. Nothing to do with comparison method.

Board-wide sweep for `drift`, `interleav`, `paired`, `a/b`, `baseline`, `benchmark`, `repeats`
found no ticket asking for repeated or interleaved measurement. **Unowned.**

## Severity

**Medium**, on the rubric's soft-blocker band: *"doable, but only via a documented workaround, a
source dive, or many extra calls"*. It is doable — a working harness exists — but only outside the
plugin, in a map-project Python producer nobody on another project has, driven through
`python.execute` and a JSON args file.

**The High reading is stated and declined.** *"Silent false-success ... the caller trusts a result
that is a lie and builds on it"* describes what actually happened here almost exactly: three
percentage figures were produced, believed, and written down, and none survived. I decline High
because the falsehood is produced by the *caller's method*, not returned by a verb — no PinWright
call reported a wrong number. That is the same line `F-generic-asset-create-verb` and
`F-spatial-swept-shape-query` draw between a bad result and a bad tool, and it is the honest one.
Whoever disagrees has a real argument and it should be argued on the ticket rather than by
re-rating quietly.

**Reach modifier declined.** Performance A/B is not an every-session path, which by the rubric
argues a bump down to Low. Declined: Low is *"pure friction — docs, discoverability, naming,
cosmetic"*, and the measured cost here was not friction but three wrong conclusions, one of them
off by 6x, recorded in a document other work then read.

severity rationale: impact=soft blocker with a working out-of-plugin workaround, High declined
because the falsehood is method-side not verb-side (Medium) x reach=rare path, bump down to Low
declined because the cost is wrong conclusions and not friction -> Medium

## Not done

Plugin source was not modified and no new measurement was taken for this filing. Every number is
cited from `Docs/map/vegetation-performance.md` on the host that produced it, with the drift table
and the invented-conclusion figures read at their line numbers rather than relayed; the
`pw_paired_ab.py` mechanics and its block-alternation plan were read out of the file itself.
`pw_paired_ab.py` was not re-run.

## History
- `#1-no-interleaved-measurement-anywhere` `OPEN` reporter — Filed from the final triage sweep of a
  vegetation performance session on host `EAContentExamples58` (UE 5.8, editor build 13:32, plugin
  commit `d8f1bc32`). Evidence: `Docs/map/vegetation-performance.md` § *Measurement hazard 8*
  (heading `:378`), drift table `:383-388` (F6 24.27 -> 19.81, F1 19.88 -> 16.31, W1 16.11 -> 13.27
  all -18%, W2 13.63 -> 10.60 at **-22%**), invented conclusions `:390-394` (-27% / -16% / -20%,
  the last re-measuring at -3.3%), p10-does-not-cover-this at `:392-394` against hazard 4 at `:168`,
  and the sentence placing the general lesson in PinWright at `:399`. **Two things in the lead I was
  handed were imprecise and are corrected here:** the headline "18% faster" is the F6/F1/W1 figure —
  W2's own drift row is **-22%**, so quoting W2's raw numbers under an 18% headline understates it;
  and only the per-species table was re-measured interleaved, so "all three were re-measured" would
  be false (`MaxPixelsPerEdge 2`'s honest interleaved counterpart is -5.8% at F6 in the lever table
  at `:337-346`, and `ResolutionLodBiasDirectional 0` does not appear there). **Plugin side,
  re-derived at HEAD:** swept `Source/` for `paired|interleav|ab_test|abTest|benchmark|baseline|
  repeats|alternat` — the only measurement-shaped hit is `performance.run_benchmark`
  (`Handlers/Debug/PerformanceHandler.cpp:860`, body `:860-907`), which has no A/B, no repeats and
  no statistics, and returns `{captured:false}` on UE 5.8 (`:894-900`, macro `:27-31`); everything
  else is unrelated comment prose. **The working implementation is map-project code, not plugin
  code:** `dev/perf/pw_paired_ab.py`, whose plan at `:93-97` is `for _ in range(repeats): for cfg in
  ("A","B"): for pose in poses` — **block alternation, not strict per-pose ABAB**, which is worth
  stating because "interleaved" invites the stronger assumption. Ask: an interleaved measurement
  verb reporting per-configuration statistics **plus the A-repeat spread** and a same-scene guard
  (`primitivesDrawn`, which jumped 95,648 -> 151,568 mid-comparison in this session,
  `vegetation-performance.md:174-176`), restoring the entry configuration on every exit path;
  failing that, a documented recipe in the plugin wiki. Cross-linked to
  `F-performance-frame-time-statistics` as the primitive this needs (neither closes the other; the
  primitive lands first), to `E-viewport-info-no-render-resolution` as one named confounder against
  this ticket's general method, and to `F-report-concurrent-engine-processes` as the sibling
  confounder interleaving does not cancel. Severity Medium; High stated and declined because the
  false conclusion is produced by the caller's method rather than returned by a verb; reach bump
  down to Low declined because the cost was three wrong conclusions, not friction.
