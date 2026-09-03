---
id: F-metasound-no-render-to-pcm
title: "No verb renders a MetaSound Source offline, so a timed multi-stage graph can only be verified structurally — never heard or measured"
status: OPEN
severity: Medium
category: feature
tags: [metasound, audio, authoring, render, offline-render, verification, timing, trigger-delay, audio.synth, audio.analysis]
encounters: 1
lastSeen: 2026-09-02T22:43:19.3167158+03:00
---

# A built MetaSound Source cannot be rendered to PCM, so its output is unverifiable

`audio.authoring.*` can build a `UMetaSoundSource` end-to-end and
`compile_metasound` will report `valid: true` with empty `diagnostics`, but
**nothing in the RPC surface turns that graph into samples.** The author's only
readbacks are structural — `describe_metasound` (nodes + edges) and
`decompile_metasound` (MSIR text). Both describe the graph the author just
wrote; neither can tell whether it *sounds* the way it was specified.

The gap bites hardest on the case MetaSounds exist for: **timed, multi-stage
graphs.** Building `MS_Reload_AR` (a magazine-out / magazine-in / bolt-release
reload staged with two chained `UE.Trigger Delay` nodes at 0.60 s each), the
question that actually needed answering was "do the three Wave Players start
0.6 s apart, or do they all fire at once?" — the exact failure a mis-wired
`Play` pin produces, and one `compile_metasound` reports as `valid: true`
either way, because a graph with all three players wired straight to
`UE.Source.OnPlay` is perfectly legal.

## What is available, and why none of it covers this

- `audio.synth.generate` / `audio.synth.audition` render **recipes**, not
  assets: they synthesize from a recipe spec, and there is no way to point
  either at an existing `UMetaSoundSource`.
- `audio.analysis.analyze` / `compare` / `decompose` operate on **SoundWaves**.
  They would answer the timing question precisely (onset positions at 0.00 /
  0.60 / 1.20 s) — but only if something first produced a wave from the
  MetaSound, and nothing does.
- `audio.play_sound_2d` and friends play into the live editor session. They
  produce no artifact to measure, need audio hardware, and are unusable from a
  headless or shared-editor context.
- `sequencer` / MRQ render the level, not a standalone source asset.

So the surface has a renderer (synth), an analyzer (analysis), and an authoring
API (authoring) — with **no edge between authoring and either of the other
two.**

## Workaround used

Verify structurally from `decompile_metasound` MSIR and treat the wire topology
as a proxy for timing. For `MS_Reload_AR` the MSIR shows exactly one wave
player fed from the play trigger and the other two fed only through the delay
chain:

```
node n8 = `UE.Trigger Delay` v=1.0 { `Delay Time`: 0.600000 }
node n9 = `UE.Trigger Delay` v=1.0 { `Delay Time`: 0.600000 }
wire $UE.Source.OnPlay -> n5.Play      // MagOut at 0.00 s
wire $UE.Source.OnPlay -> n8.In
wire n8.Out -> n6.Play                 // MagIn  at 0.60 s
wire n8.Out -> n9.In
wire n9.Out -> n7.Play                 // Bolt   at 1.20 s
```

That is a real check — it rules out the simultaneous-trigger bug, and MSIR is
the right tool for it. What it cannot do is confirm the delays *fire* at the
stated times, that the mixer sums all three without clipping, that a stage is
not truncated by an early `OnFinished`, or that the `UE.RandomFloat` → `Pitch
Shift` wiring produces audible per-shot variation at all. Every one of those is
a measurement, and the surface offers no way to take it. This forces
"structurally correct" to be reported where "verified" was asked for.

## What it should do

Add an offline render verb, e.g. `audio.authoring.render_metasound`:

- in: `assetPath`, `durationSeconds`, optional `sampleRate`, optional graph
  input literals to set before rendering;
- out: a `USoundWave` written to a caller-named content path (feeding
  `audio.analysis.*` directly), and/or a `.wav` under `Saved/`, plus inline
  summary stats (peak, RMS, duration).

That single edge closes the loop: build with `authoring`, render, then measure
onsets with `audio.analysis.analyze` to prove a staged graph is staged. The
engine already renders sources offline for cook/preview, so the machinery
exists; only the RPC does not.

An `audio.synth.audition`-shaped variant taking an `assetPath` instead of a
recipe would be an acceptable smaller first step.

severity rationale: impact=hard blocker with no workaround for verifying rendered output (structural proxy only, doable but never a measurement) x reach=every timed or multi-stage MetaSound build, a core use of the authoring surface -> Medium

## History
- `#1-filed` `OPEN` reporter — Filed while building seven FPS MetaSound Sources under `/Game/FPS/Audio/MetaSounds/` via `audio.authoring.*`. All seven compiled `valid: true` with empty diagnostics and persisted to disk, but the one question the task actually asked — whether `MS_Reload_AR`'s three Wave Players are staggered 0.6 s apart by two chained `UE.Trigger Delay` nodes rather than firing simultaneously — has no verb that can answer it by measurement. `compile_metasound` returns `valid: true` for the mis-wired simultaneous graph just as readily as for the correct one, so it is not a check on staging at all. `audio.synth.*` renders recipes and cannot be pointed at a `UMetaSoundSource`; `audio.analysis.*` measures SoundWaves and would answer the timing question exactly (onsets at 0.00/0.60/1.20 s) but nothing produces the wave to feed it; `audio.play_sound_2d` plays into the live session and leaves no artifact. Worked around by reading `decompile_metasound` MSIR and confirming only `n5.Play` is wired to `$UE.Source.OnPlay` while `n6.Play` comes from `n8.Out` and `n7.Play` from `n9.Out` (delays 0.600000 each) — sound topology evidence, but a proxy, not a measurement: it cannot confirm the delays fire at the stated times, that the Mono-3 mixer sums without clipping, that no stage is truncated by an early `OnFinished`, or that `UE.RandomFloat` → `Pitch Shift` yields audible variation. Requests `audio.authoring.render_metasound {assetPath, durationSeconds, sampleRate?}` emitting a `USoundWave` (and/or a `Saved/` `.wav`) plus peak/RMS/duration, which would let `audio.analysis.analyze` close the build→verify loop; an `audio.synth.audition` variant accepting an `assetPath` would be a smaller acceptable first step.
- `#2-blocks-review` `OPEN` reporter — Second encounter, from the reviewing side rather than the authoring side, and it blocks the single highest-value check in an audio review. Reviewing `/Game/FPS/Audio/` (AUDIO stream, `Docs/fps/reviews/audio-review-01.md`) the rubric row "per-shot variation audible as pitch/gain differences between two consecutive renders" is, by construction, a *measurement of two renders* — and there is no way to take it. `audio.analysis.analyze { assetPath }` accepts a `USoundWave` only; pointed at `/Game/FPS/Audio/MetaSounds/MS_Fire_AR` it cannot be used, and nothing produces a wave from that source. So the strongest statement a critic can make is structural: `decompile_metasound` MSIR shows `node n11 = UE.RandomFloat { Min: -1.2, Max: 1.2 }` with `wire $UE.Source.OnPlay -> n11.Next`, `wire n11.Value -> n6.Pitch Shift` and `-> n7.Pitch Shift`, plus `n12` (±0.7 st -> sub) and `n13` (0.82-1.0 -> mixer Gain 0/1/2). That proves the wiring exists; it cannot show the range is *audible*, that mech and body sharing one value (both fed from `n11`) reads as correlated, or that at 700 RPM the ear locks onto the unchanging source waves anyway. The same gap also blocked checking the composite balance: the three AR layers measure RMS -29.0 / -24.86 / -11.92 dB and the graph gives them equal mixer gains, so the summed shot is almost certainly sub-dominated — but "almost certainly" is as far as MSIR plus per-wave analysis can get, and a render would settle it in one call. Note the asymmetry this creates in review: every *wave* claim in the build report could be re-derived and several were wrong, while every *graph* claim had to be taken on topology alone. Restating the ask from `#1`: `audio.authoring.render_metasound { assetPath, durationSeconds, sampleRate?, inputs? }` emitting a `USoundWave` (or a `Saved/` `.wav`) that `audio.analysis.analyze` and `audio.analysis.compare` can consume; two renders of the same source plus one `compare` call is the whole per-shot-variation check.
