---
id: B-synth-schema-advertises-unrenderable-ranges
title: "audio.synth.describe_schema publishes parameter ranges the render path rejects — ringmod.rateHz and filter.cutoffHz disagree between the spec table and the DSP"
status: DONE
severity: Medium
category: bug
tags: [audio, synth, describe_schema, validation, single-source-of-truth, contract]
encounters: 1
lastSeen: 2026-08-18T00:00:00+03:00
---

# The schema and the renderer disagree about what is legal

`audio.synth.describe_schema` renders its parameter ranges from the spec table in
`PwSynthRecipe.cpp`, which is documented as the single source of truth. Two effect
parameters are enforced more narrowly by the DSP than the table advertises, so a
recipe that is valid per the published schema is rejected at render time.

| parameter | published range | actually enforced | where |
|---|---|---|---|
| `ringmod.rateHz` | `0.1 .. 20000` | `10 .. 10000` | `PwFxChainB.cpp:826` |
| `filter.cutoffHz` | `.. 20000` | `.. 0.45 * sampleRate` | `PwFxChainA.cpp:286` |

Both narrowings are **correct and deliberate**, and should not simply be removed.
`Audio::FRingModulation` silently clamps its carrier outside roughly 10-10000 Hz, and
`Audio::FBiquadFilter` silently clamps cutoff to about `[5 Hz, 0.45 * SampleRate]`.
Reporting back a requested value the filter did not use is the defect class
`rpc-design.md` §1 exists to prevent, so rejecting is right. The bug is that the
schema does not say so.

## Why this matters more than it looks

No preset library ships with the synth namespace. `describe_schema` plus the cookbook
(`docs/wiki-src/audio.synth.cookbook.md`) is the **entire** cold-start path for an
agent authoring a recipe. An agent that trusts the published range and picks
`rateHz: 5` gets an error on its first call, from a schema it was told is
authoritative.

## The two cases need different fixes

**`ringmod.rateHz` is a plain range error.** The spec-table row should publish
`10 .. 10000` to match the enforcement. One-line fix, and `describe_schema` becomes
true again.

**`filter.cutoffHz` cannot be fixed by a static range**, because the ceiling depends
on the render's `sampleRate` — a cross-field constraint no single spec-table row can
express. `FPwSynthKindSpec` already carries a `Constraint` string for exactly this
purpose (the `modal` row uses a `Validator` hook for its equal-length arrays). Publish
the rule there so it reaches `describe_schema` output, e.g.
"cutoffHz must be below 0.45 * sampleRate; at 44.1 kHz the effective ceiling is
19845 Hz, below the published 20000".

## Sweep for the same class before closing

These two were found while writing the cookbook against the implementation, not by a
test — which means the general case is unguarded. Before calling this DONE, grep every
`PwFx*.cpp` and `PwGen*.cpp` rejection against its spec-table row and account for each
divergence as fixed, deliberately-published, or absent. A contract test asserting that
every published range is renderable at the default sample rate would make the whole
class unrepresentable, and is the better fix if it is cheap.

## History

- 2026-08-18 - Filed OPEN. Found while authoring `audio.synth.cookbook.md` against the
  real generator and effect sources rather than against the plan; the cookbook
  documents the effective windows so it does not repeat the schema's claim.
- `#2-ranges-fixed-constraints-published` `IN-REVIEW` developer — Still reproducible: `RingModParams()` published `rateHz` 0.1..20000 against `PwFxChainB` refusing outside 10..10000, and `filter` had no `constraint`. Changes in `Source/PinWright/Private/AudioGen/PwSynthRecipe.cpp`: `ringmod.rateHz` row is now 10..10000 (parse-time refusal with a field path; behaviour change in CHANGELOG), and `constraint` strings are published for `filter` (cutoffHz <= 0.45 * sampleRate, with the 48 k / 44.1 k / 8 k ceilings), `eq` (lowHz/midHz/highHz same window), `modal` (modeFreqsHz < Nyquist appended to the existing equal-length rule), `noise` (lowCutHz < highCutHz), `granular` (positionStart <= positionEnd) and `pitchshift` (formantPreserve must be false). Sweep of every `PwFx*.cpp` / `PwGen*.cpp` / `PwSynthModulation.cpp` rejection: delay feedback/timeMs, playbackRate, grainMs/densityHz, formant f0/formantShift/voicing/breathiness, chorus/flanger/phaser/compressor/gain/width reads all match their rows (absent); am/ring depth > 1 is already stated in the shared modulation row's meaning (published); the formant all-formants-above-0.45*SR refusal is unreachable inside the published ranges (absent); width-in-layer is published as `masterOnly`. Found one silent-clamp divergence of a different class (noise band limits above 0.45 * sampleRate) and filed it as `B-synth-noise-band-limit-silently-clamps`. Contract test that makes the class unrepresentable: `PinWright.audio.synth.render.PublishedParamRangesRenderAtDefaultRate` drives every numeric row of every generator/effect kind (except the asset-path kinds granular/sample/convolve) to its published min and max, every enum to each token and every boolean to both values, renders at 48 kHz, and fails on any rejection from a kind with no published `constraint`; it would have caught `ringmod.rateHz = 0.1` and `pitchshift.formantPreserve = true`. Wiki: cookbook "render time" paragraph rewritten; `audio.synth.describe_schema` section points at `constraint`.
- `#3-review-fixes` `IN-REVIEW` developer — Review finding: `PublishedParamRangesRenderAtDefaultRate` accepted any rejection from any row of a constrained kind, so e.g. a new `filter.q` rejection would pass under the cutoffHz constraint. The escape now requires the kind's `constraint` text to name the rejected row (`FString(Kind.Constraint).Contains(Row.DisplayName, ESearchCase::CaseSensitive)`), in `Source/PinWright/Private/Tests/Media/TestPwSynthRender.cpp`; every currently constrained row (noise lowCutHz/highCutHz, modal modeFreqsHz, filter cutoffHz, eq lowHz/midHz/highHz, pitchshift formantPreserve) is named in its kind's string, so the non-vacuity check still holds. NIT: the modal constraint now says the 20000 Hz row maximum is reachable at "sampleRate above 40000" (Nyquist at exactly 40000 is 20000, which is rejected) instead of "40000 or above" (`Source/PinWright/Private/AudioGen/PwSynthRecipe.cpp`). Still unrun: the sweep and `describe_schema` sections tests need the build.
- `#4-run1-fixes` `IN-REVIEW` developer — Run 1: `PinWright.audio.synth.render.PublishedParamRangesRenderAtDefaultRate` failed on `formant.voicing = 0 (min)`, which `PwGenFormant.cpp` rejects with INVALID_PARAMS because `breathiness` defaults to 0 and both at 0 make a silent source. That rule was unpublished, which is a real instance of this ticket's defect and was missed by the #2 audit, not a test error. `formant` now publishes `constraint` ("voicing and breathiness must not both be 0 ... voicing 0 needs a breathiness above 0") in `Source/PinWright/Private/AudioGen/PwSynthRecipe.cpp`; the sweep's named-row escape matches it on `voicing`. Cookbook render-time paragraph and the CHANGELOG constrained-kind list gained `formant`.
- `#5-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit 1a89f8ef). `PinWright.audio.synth.render.PublishedParamRangesRenderAtDefaultRate` passed non-skipped in run3/full. It drives every numeric row of every generator and effect kind to its published min and max, plus every enum token and both booleans, at 48 kHz. Any rejection fails unless the kind's `constraint` names that row. `ringmod.rateHz` is now published 10..10000. `filter.cutoffHz` (0.45 * sampleRate) and the rules for eq, modal, noise, granular, pitchshift and formant are published as `constraint`. The sweep was done; the noise silent clamp was split out as `B-synth-noise-band-limit-silently-clamps`. Coverage limit: `PinWright.audio.synth.describe_schema.Sections` is skip-marked (`no-page-boundary`), so no passing test asserts `constraint` in the RPC output. That emission is unchanged pre-existing code (`AudioSynthSchemaHandler.cpp:190-192`) and was verified by source read.
