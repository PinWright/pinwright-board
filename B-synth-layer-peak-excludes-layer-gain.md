---
id: B-synth-layer-peak-excludes-layer-gain
title: "audio.synth.generate `render.layers[].peakLinear` is measured BEFORE the layer's own `gainDb`, so the one number the cookbook says to balance layers with does not move when you balance them"
status: OPEN
severity: High
category: bug
tags: [audio, synth, generate, render-report, gainDb, misleading-measurement, wiki-mismatch]
encounters: 5
lastSeen: 2026-09-03T01:30:00Z
---

# `render.layers[].peakLinear` ignores the layer's `gainDb`

## Symptom

`audio.synth.generate` and `audio.synth.patch` report a per-layer render block:

```json
"layers":[{"layer":0,"framesMixed":8640,"peakLinear":1.0706}, ...]
```

`peakLinear` is measured upstream of the layer's `gainDb` trim. Changing `gainDb` — the one
knob the cookbook names for inter-layer balance — leaves the reported peak **bit-identical**,
even for a 28 dB change, while the mix bus plainly moves.

## Repro (one session, UE 5.8 / UE 5.7 editor, EAContentExamples58, gateway 27145)

Candidate `c1_82cf`, a 4-layer concrete-footstep recipe. Then `audio.synth.patch` with
`replace` ops on four `gainDb` fields at once:

| layer | gainDb before | gainDb after | delta | `peakLinear` before | `peakLinear` after |
|---|---|---|---|---|---|
| 0 (`modal`)  | -3  | -12 | -9 dB  | 1.0706 | 1.0706 |
| 1 (`noise`)  | -12 | -10 | +2 dB  | 0.6524 | 0.6627 * |
| 2 (`noise`)  | -16 | +12 | +28 dB | 0.0161 | 0.0161 |
| 3 (`noise`)  | -14 | -10 | +4 dB  | 0.4791 | 0.4791 |

\* layer 1 is the only one that moved at all, and it moved because the same patch also changed
its `lowCutHz` from 2000 to 1200 — a generator change, not a gain change.

That the gains *were* applied is not in doubt: the same patch moved
`render.peakDbBeforeNormalize` from -3.28 to -6.99 and swung the analysis bands hard
(`hz20_150` 0.2167 -> 0.0117, `hz4000_8000` 0.1519 -> 0.4556). Only the per-layer peak is blind
to it. Reproduced again on candidate `c8_be69` -> `c11_6a9f`: layer 2 moved -10 dB -> -22 dB and
its `peakLinear` stayed 0.3989 in both reports.

## What the documentation promises

`audio.synth.cookbook`, "Read this part first", ends its `gainDb` table with the instruction
this report is about:

> Absolute level is master `normalize`'s job; layer `gainDb` is for balance *between* layers,
> and you set it by rendering and reading the peak rather than by reasoning.

`audio.synth.generate`'s own wiki page sells the same number as the load-bearing one:

> per-layer frames mixed (straight off the mix bus, so a layer that contributed nothing reads 0
> rather than looking identical to one that landed)

"straight off the mix bus" is exactly what it is not. A layer at `gainDb: -96` — inaudible,
contributing nothing — reports the same healthy `peakLinear` as the same layer at `0`, which is
the precise confusion the sentence claims to prevent.

## Impact

`High`. The documented balance loop ("render and read the peak") cannot converge: the reading
never responds to the adjustment, so an author either concludes the patch did not apply and
re-sends it, or abandons the report and balances by eye off the spectrogram. It is a misleading
measurement rather than a corrupting one — the mix itself is correct — but it silently defeats
the only quantitative feedback the subsystem offers for layer balance, on a namespace whose
whole pitch is "iterated on against measured numbers".

Cost in this session: three wasted iterations on the first sound before the pattern was
recognised, then hand-balancing all 18 foley waves off `analysis.spectral.bands` proportions
instead.

**Workaround:** treat `peakLinear` as a *generator* diagnostic only — useful to catch a
generator that is far weaker than expected (a `blue` noise band-limited to 3-8 kHz reports
0.015, i.e. it needs roughly +30 dB before it is audible next to a `white` layer) and to
confirm a layer rendered at all. For actual balance, patch `gainDb` and read
`analysis.spectral.bands` and `render.peakDbBeforeNormalize`, which do respond.

**Fix:** measure the peak after the layer's `gainDb` (and ideally after its `fx` chain and pan
law), which is what "straight off the mix bus" describes. If the pre-gain value is deliberate,
publish both — `peakLinear` (post-gain, the mix contribution) alongside something like
`generatorPeakLinear` — and correct the two wiki sentences above, because as written they both
describe the post-gain number.

## History
- `#1-filed` `OPEN` reporter — Hit while synthesizing 18 FPS foley waves into `/Game/FPS/Audio/Waves/Foley/` on EAContentExamples58. An `audio.synth.patch` on candidate `c1_82cf` replaced `gainDb` on all four layers (-9, +2, +28 and +4 dB) and every `render.layers[].peakLinear` came back identical to four decimal places (1.0706 / 0.6524 / 0.0161 / 0.4791), while the same response's `render.peakDbBeforeNormalize` moved -3.28 -> -6.99 and `analysis.spectral.bands.hz4000_8000` moved 0.1519 -> 0.4556, proving the gains had applied. The single exception, layer 1 at 0.6524 -> 0.6627, is explained by that layer's `lowCutHz` also changing 2000 -> 1200 in the same patch. Reproduced on a second recipe (`c8_be69` -> `c11_6a9f`, layer 2 at -10 dB -> -22 dB, `peakLinear` 0.3989 both times). The misleading pages are `audio.synth.cookbook` ("you set it by rendering and reading the peak rather than by reasoning") and `audio.synth.generate` ("per-layer ... peak ... straight off the mix bus, so a layer that contributed nothing reads 0 rather than looking identical to one that landed") — a `gainDb: -96` layer contributes nothing and still reports its full generator peak, which is the exact failure that sentence promises is impossible. Rated High as a misleading measurement that defeats the namespace's documented quantitative iteration loop without corrupting output. Workaround used for the remaining 17 waves: balance off `analysis.spectral.bands` and `render.peakDbBeforeNormalize`, and read `peakLinear` only as a generator-strength diagnostic.
- `#2-confirmed` `OPEN` reporter — Independently reproduced on a different recipe family (bullet-impact and explosion waves into `/Game/FPS/Audio/Waves/Impacts/`, same editor session, gateway 27145). `audio.synth.patch` on candidate `c9_3e83` -> `c13_5187` replaced `gainDb` on three `modal` splinter layers by -22, -22 and -22 dB (-14 -> -36, -17 -> -39, -20 -> -42) and all three `peakLinear` values came back bit-identical (1.4806 / 1.3753 / 1.1649), while the same response moved `render.peakDbBeforeNormalize` -15.20 -> -16.12 and `analysis.spectral.bands.hz500_1500` 0.6447 -> 0.0271, so the gains had plainly applied. Reproduced a third time on `c10_2793` -> `c14_693a` (four layers, -15/-16/-16/-16 dB, all four peaks identical). Confirms the report's diagnosis and adds a second-order trap the workaround should name: because `peakLinear` is the raw generator peak and `noise` is RMS-referenced, a band-limited `noise` layer reports a peak an order of magnitude below a `modal` layer of the same audible weight — a `pink` layer band-limited 1200-7000 Hz read 0.0483 against a `modal` bank's 0.7052, so the noise needed roughly +17 dB relative trim to sit 8 dB *under* the bank. Balancing by the published number therefore does not merely fail to converge, it converges backwards for noise-versus-modal pairs: three of my surfaces (concrete B, dirt A, dirt B) rendered as almost pure low-frequency thud with `bands.hz20_150` at 0.97 / 0.96 / 0.80 because the noise spray layers were inaudible at gains that looked balanced. Corrected by computing each layer's true contribution as `peakLinear * 10^(gainDb/20)` by hand and balancing on that, which is the arithmetic the report proposes the server should do.
- `#3-minimal-two-recipe-repro` `OPEN` reporter — Third independent hit, from the weapon-layer work in `/Game/FPS/Audio/Waves/Weapons/`, and adding a **minimal controlled repro** that isolates `gainDb` as the only variable and supplies the in-response control proving the gain itself is applied. Two `audio.synth.generate` calls, `analyze:false`, recipes byte-identical except for `layers[0].gainDb`: a single `osc` sine at 500 Hz, 200 ms, trapezoid `ampEnvelope`, `master.normalize {mode:peak, target:-1}`, `seed:2`. With `gainDb: 0` -> candidate `c52_b696`, `render.peakDbBeforeNormalize: -3.01`, `render.layers[0].peakLinear: 1`. With `gainDb: -40` -> candidate `c53_b6f0`, `render.peakDbBeforeNormalize: -43.01`, `render.layers[0].peakLinear: **1**`. The bus level moved by exactly the 40 dB requested, so the gain is unquestionably applied to the render; the per-layer peak did not move at all, not even in the fourth decimal. Because both takes are single-layer, nothing else (pan law, summation with a sibling, normalizer headroom) can be blamed for absorbing the difference. Cost in practice: I was balancing a three-layer indoor reverb tail whose `modal` room-resonance layer reported `peakLinear: 1.9218` at `gainDb: -16`; I pulled it to `gainDb: -28` and the report came back `peakLinear: 2.0397`, i.e. *higher* after a 12 dB cut, which reads as the cut having failed. The only way I could confirm the cut had actually landed was the unrelated `analysis.spectral.bands` split, where `hz20_150` fell from 0.7736 to 0.0151 — a different family, measured for a different purpose, which happens to expose the truth. Reinforces the ask in `#1`: either measure the per-layer peak after `gainDb` (and say whether pan is included), or rename the field and add a second post-gain `contributionPeakLinear`, and fix `audio.synth.generate`'s "straight off the mix bus, so a layer that contributed nothing reads 0 rather than looking identical to one that landed" — under the current behaviour a layer at -40 dB reads *exactly* identical to one at 0 dB, which is the precise failure that sentence promises the field prevents.
- `#4-ambience-beds-noise-vs-noise` `OPEN` reporter — Fourth independent hit, this time on **ambience loop beds** in `/Game/FPS/Audio/Waves/Ambience/` (same editor, gateway 27145), and it adds a `noise`-versus-`noise` case that `#2` only covered across generator families. Every one of my three beds needed a full rebalance round driven entirely by `analysis.spectral.bands`, because the per-layer peaks are not merely unresponsive to `gainDb` — for coloured noise they are actively inverted relative to audibility. In candidate `c80_df31` (12 s coastal wind) the four layers reported `peakLinear` 1.1384 / 0.0943 / 0.0278 / 0.1168: a `brown` layer band-limited 25-900 Hz reads 12x higher than a `pink` layer band-limited 400-4000 Hz that should sit *above* it in the mix, purely because `brown` is RMS-referenced and overshoots in peak. Balanced by the reported numbers, the render came back with `bands.hz20_150` at 0.8851 and `hz1500_4000` at 0.0002 — a pure rumble with the entire wind gone. The same trap fired again on the sea bed (`c97_d19c`, `hz20_150` 0.5721) and on the generator bed (`c102_9b05`, `hz20_150` 0.9775, i.e. a 50 Hz hum with no cylinder body and no rattle). Three beds, three wasted renders, one cause. What finally worked is the arithmetic `#2` names: compute each layer's true contribution as `peakLinear * 10^(gainDb/20)` by hand, then trim on that — e.g. wind layer 0 at `1.1384 * 10^(-4/20) = 0.72` against layer 1 at `0.0943 * 10^(-10/20) = 0.030`, a 27 dB gap that the four reported peaks do not show. Adds to the ask: whatever the field ends up measuring, the `noise` generator's RMS reference should be stated next to it, since a `brown` and a `pink` layer at the same audible level differ by more than 20 dB in the published peak.
- `#5-round-robin-variants-and-a-second-order-cost` `OPEN` reporter — Fifth hit, building the Build 04 weapon round-robin variants (`Docs/fps/reviews/audio-review-03.md` §5 item 3) in `/Game/FPS/Audio/Waves/Weapons/`. Nothing new about the mechanism — `#3`'s minimal repro already settles it — but it names a **second-order cost that only shows up when matching an existing asset rather than authoring a fresh one**, and that is the case the round-robin ask creates. Matching a sibling means driving six band fractions onto measured targets simultaneously, so every iteration has to attribute a band change to the right layer. With `peakLinear` blind to `gainDb` there is no per-layer contribution number to attribute against, and `analysis.spectral.bands` — the workaround the earlier encounters land on — is a **mix-bus measurement**: it says the 4-8 kHz band moved, never which of the four layers moved it. Concretely, on `SW_Fire_Pistol_Mech_B` a change of `modeGainsDb` on the mode bank and a change of the noise layer's `highCutHz` both move `hz4000_8000`, and I twice mis-attributed a band swing to the mode gains when it came from the noise band — burning renders `c105`-`c108` before I fixed the seed and changed one thing at a time. Total for four variants: **21 `generate` calls**, of which I judge roughly a third were spent re-deriving per-layer contribution that a correct `peakLinear` would have published outright. Two additions to the ask: (a) the post-gain number is worth more than the pre-gain one *specifically* because it is the only per-layer quantity in the response — everything else is bus-level, so today a multi-layer recipe has no per-layer feedback at all once `gainDb` is in play; (b) whichever number is published, changing the recipe `seed` re-rolls a `noise` exciter and moves a `modal` layer's peak by up to 3.6 dB on an otherwise identical recipe (`c109` 1.9958 vs `c110` 1.4276 vs `c112` 1.7525, same mode bank), so any iteration loop that varies the seed is comparing two things at once — worth a sentence in the cookbook beside the balancing advice.
