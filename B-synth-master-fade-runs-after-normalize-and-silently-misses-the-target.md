---
id: B-synth-master-fade-runs-after-normalize-and-silently-misses-the-target
title: "master.fadeInMs is applied AFTER normalize, so a fade that overlaps the peak leaves the render below its normalize target while the render report still claims the gain that would have hit it"
status: DONE
severity: Medium
category: bug
tags: [audio, synth, generate, normalize, fade, peak, misleading-measurement, transient]
encounters: 1
costly: 1
lastSeen: 2026-09-03T01:30:00Z
---

# A master fade silently eats the normalize target

## Symptom

`audio.synth.describe_schema` documents the chain as

    recipe -> layers[] -> ... -> stereo mix bus -> master.fx[] -> normalize -> fades

so `master.fadeInMs` / `fadeOutMs` run **after** the normalizer. For any sound whose peak sample
lies inside the fade window — which is every impact, footstep, gunshot mech click and body, i.e.
anything with `attackMs: 0` — the fade attenuates the very sample the normalizer just placed at
the target, and the render lands below it.

Nothing in the response says so. `render.normalize` reports the gain it applied against the
pre-fade level, and `analysis.technical.peakDb` reports the post-fade truth, in the same object,
with no reconciliation. Both numbers are individually correct; read together they contradict, and
only the second is the asset you get.

## Repro (UE 5.8, EAContentExamples58, Build 04 audio)

Candidate `c99_a432`, a 6-layer AR body: `master.normalize {"mode":"peak","target":-1}`,
`master.fadeInMs: 0.3`, `fadeOutMs: 6`. Peak sample at t=0 (broadband onset, `attackMs: 0`).

```json
"render":{"normalize":{"measured":true,"gainDb":1.68,"inputDb":-2.68}}   // -2.68 + 1.68 = -1.00
"analysis":{"technical":{"peak":0.8096,"peakDb":-1.83}}                  // actual: -1.83
```

0.83 dB gone. A 0.3 ms fade at 48 kHz is 14 samples; the peak was inside them.

The same recipe with `master.fadeInMs: 0` and the 0.3 ms ramp moved into each layer's own
`ampEnvelope` (`[{"timeMs":0,"value":0},{"timeMs":0.3,"value":1,"curve":"exp"}, ...]`) — so the
ramp is upstream of the normalizer instead of downstream — rendered `peakDb: -1.00` exactly, with
`startDiscontinuity: 0` preserved. Candidate `c100_30ab`. That is the workaround and it is
exact, but it is only findable by noticing the contradiction first.

## Why it matters here

Peak-normalising a whole family to one level is the standard move in this library — the weapon
bodies are all `-1.0 dBFS`, the mech layers `-6.0`, the footsteps `-1.5` — and the review
process checks that level on the exported asset. A wave that silently lands 0.8 dB low reads as
a level outlier in the next audit, with the recipe apparently asking for the right thing.
`master.fadeInMs` is also the obvious remedy for a non-zero `startDiscontinuity`, so a caller
fixing one measured defect quietly introduces another.

## Asks

- **Reconcile the two numbers in the response.** Either publish the post-fade peak inside
  `render.normalize` (a `outputDb` beside `inputDb`/`gainDb`), or emit a warning when the applied
  fade attenuates the sample the normalizer keyed on. One line — "fadeInMs 0.3 covers the peak
  sample; output peak -1.83 dBFS, target -1.00" — turns a silent miss into a decision.
- **Document the interaction on `audio.synth.generate`.** The topology string states the order,
  but nothing joins it to the consequence, and the normalize wiki text ("Read
  `render.normalize.measured` before `render.normalize.inputDb`: the default is not a level")
  warns about a different trap entirely.
- Consider whether a fade should be measured *before* normalize for peak mode, so the target is
  honoured by construction. That changes existing renders, so it is the weakest of the three.

## History
- `#1-filed` `OPEN` reporter — Hit building the Build 04 weapon round-robin variants
  (`Docs/fps/reviews/audio-review-03.md` §5 item 3). `SW_Fire_AR_Body_B` needed to match the
  original's `-1.0 dBFS` exactly; the first converged take was 0.83 dB under it and the recipe
  looked correct. Found by comparing `render.normalize` arithmetic against
  `analysis.technical.peakDb` in the same response.
- `#2-output-level-published` `IN-REVIEW` developer — Took ask 1 and ask 2; ask 3 (measure before fades) declined because it changes every existing render. `FPwRenderReport` gains `OutputPeakDb`, the peak of the final buffer after normalize, fades and clamp (`Source/PinWright/Private/AudioGen/PwSynthDsp.{h,cpp}`). `BuildRenderReport` in `Source/PinWright/Private/Handlers/Audio/AudioSynthGenerateHandler.cpp` now publishes `render.normalize.outputDb` in peak mode beside `inputDb`/`gainDb`, and a `render.normalize.warning` naming fadeInMs/fadeOutMs, the output peak, the target and the shortfall plus the ampEnvelope remedy when the output lands more than 0.01 dB under the target (generate and patch both route through it). Documented on the `audio.synth.generate` wiki section, in the cookbook ("Master fades run after normalize"), and in the `describe_schema` topology string. Test: `PinWright.audio.synth.generate.FadeOverPeakReportsOutputLevel` (100 Hz sine with a 20 ms decay and a 10 ms fade-in: outputDb present and < -2 dBFS while inputDb + gainDb still reconcile to -1, warning names fadeInMs and ampEnvelope; unfaded control lands at -1 +/- 0.02 with no warning).
- `#3-review-fixes` `IN-REVIEW` developer — Re-checked acceptance after the handoff flagged "fades not applied before normalize": ask 3 is phrased "Consider whether", and the ticket itself ranks it the weakest because it changes every existing render, so declining it is within acceptance. Asks 1 (`render.normalize.outputDb` plus the shortfall `warning`) and 2 (generate wiki, cookbook, topology string) remain the deliverable. No code change in this round; the reporter's ampEnvelope workaround is named in the warning text.
- `#4-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit 1a89f8ef). `PinWright.audio.synth.generate.FadeOverPeakReportsOutputLevel` passed non-skipped in run3/full. Ask 1: a 10 ms fade over the peak publishes `render.normalize.outputDb` below -2 dBFS. `inputDb + gainDb` still reconciles to -1, and `render.normalize.warning` names fadeInMs and the ampEnvelope remedy. The unfaded control lands at -1 +/- 0.02 with no warning. Ask 2 is checked in the doc text at HEAD. `docs/wiki-src/audio.synth.md` (generate) explains `outputDb` and why the fade misses. `audio.synth.cookbook.md` has the paragraph "Master fades run after normalize". `describe_schema` topology carries the order. Ask 3, measuring fades before normalize, was declined because it changes every existing render. The ticket worded it "Consider" and ranked it weakest, so the decline is within acceptance.
