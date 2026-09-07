---
id: F-rpc-audio-normalize-existing-wave
title: "No gain-only path for an existing SoundWave: the only route to normalize is a `sample` layer, which is mono, so a level change silently destroys the wave's stereo image"
status: OPEN
severity: Medium
category: feature
tags: [audio, synth, sample, normalize, soundwave, stereo, mono-collapse, silent-data-loss]
encounters: 1
costly: 1
lastSeen: 2026-09-03T00:50:00Z
---

# Changing an existing wave's level costs its stereo image

## Symptom

There is no verb that rescales the PCM of an existing `USoundWave`. `master.normalize` is a
recipe field, so reaching it for audio that already exists means wrapping the asset in a
`sample` layer and re-exporting over itself:

```json
{"durationMs": 640, "layers": [{"generator": {"kind": "sample",
   "params": {"sourcePath": "/Game/FPS/Audio/Waves/Ambience/SW_Amb_RopeSlap_01"}}}],
 "master": {"normalize": {"mode": "lufs", "target": -21}}}
```

This is exact in every respect the analysis measures **except stereo**: duration, frames,
transient count, crest, band split and spectral peaks all round-trip unchanged. But every
generator is mono and `sample` is documented as "the source asset's own level, mono-collapsed",
so the layer's pan is the only placement available and the result is `correlation: 1,
width: 0, monoCompatibility: 1` regardless of what went in.

Measured on four Build 01 waves renormalised this way:

| wave | correlation before | after | width before | after |
|---|---|---|---|---|
| `SW_Amb_RopeSlap_01` | 0.8106 | 1 | 0.2576 | 0 |
| `SW_Reload_Bolt`     | 0.9522 | 1 | 0.1354 | 0 |
| `SW_Impact_Wood_B`   | 0.9824 | 1 | 0.0855 | 0 |
| `SW_Impact_Concrete_A` | 0.9920 | 1 | 0.0594 | 0 |

Nothing reports it. `audio.synth.export`'s verification compares per-channel RMS and peak
against the **candidate**, which is already mono, so both channels match and
`verification.pass` is `true`. `audio.analysis.audit_folder` has no stereo-image check and
swept all 63 waves with `flagged: 1` for an unrelated DC offset.

## Why the workarounds do not work

- Two `sample` layers panned hard L/R read the *same* mono collapse, so correlation stays 1.
- Giving one of them a `sourceStartMs` offset decorrelates them, but for a loop it relocates
  the seam into the middle of the file — trading a measured defect for an unmeasured one, since
  `startDiscontinuity`/`endDiscontinuity` only look at the ends.
- A master `width` effect cannot help: there is no side signal left to widen.
- `audio.authoring.set_sound_wave_properties`' `volume` is a playback multiplier, not payload,
  so it does not move the `peakDb` / `integratedLufs` a loudness review measures.
- `audio.analysis.to_recipe` resynthesises from a fit — a redesign, not a level change.

## Impact

`Medium`, and sharply asymmetric: harmless on material that is already near-mono
(`SW_Impact_Concrete_A`, width 0.0594 -> 0) and destructive on anything deliberately wide.
It lands hardest on ambience beds, which are the assets most likely to be both wide and in need
of a loudness pass. In this encounter `SW_Amb_Sea_Loop` (correlation 0.9285, width 0.165) was
spared only because its original recipe happened to still be resident in the candidate registry
and its fix was reachable as a two-op `patch`; had it needed the `sample` route, a 14 s stereo
sea bed would have shipped in mono with every metric green.

## Ask

Either of:

- **A payload gain verb** — `audio.authoring.set_sound_wave_gain {assetPath, mode: "peak"|"lufs",
  target}` that rescales the existing PCM in place, per channel, leaving the channel count and
  image alone. This is the operation actually wanted whenever a review flags a loudness outlier,
  and it is strictly safer than a re-render because it cannot change anything but level.
- **A stereo-preserving `sample` generator** — `channelMode: "preserve"` on the `sample` params,
  bypassing pan for a stereo source. Larger change, since it breaks the "every generator is
  mono" invariant the namespace is built on.

Independently, and cheaply: `audio.synth.export` should warn when the candidate is mono-sourced
and the asset it overwrites was not, and `audio.analysis.audit_folder` should carry a
stereo-image check so a collapse is visible to the sweep that is supposed to catch it.

## History
- `#1-filed` `OPEN` reporter — Hit fixing the Build 03 loudness and normalisation defects in
  `Docs/fps/reviews/audio-review-02.md` §5 (D14, new defect 5). Four Build 01 waves needed level
  changes only — `SW_Impact_{Concrete,Wood}_{A,B}` from -3.0 to -1.0 dBFS peak to match the impact
  family, `SW_Reload_Bolt` -14.08 -> -21 LUFS and `SW_Amb_RopeSlap_01` -27.44 -> -21 LUFS toward the
  page median. None of their recipes survived (Build 01 predates the session's candidate registry),
  so the `sample`-layer route was the only one available. It preserved every metric the review
  scores except stereo, which went to `correlation 1 / width 0` on all four with
  `verification.pass:true` and a clean `audit_folder`. Filed rather than worked around because the
  next caller with a wide ambience bed and a loudness note has no way to see the trade before
  paying it.
