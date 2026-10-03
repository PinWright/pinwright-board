---
id: F-audio-no-mono-soundwave-path
title: "No verb in the plugin can produce a 1-channel USoundWave: audio.synth.export and create_sound_wave_from_pcm both force stereo, so every wave a mono MetaSound consumes costs 2x memory"
status: DONE
severity: Medium
category: feature
tags: [audio, synth, export, soundwave, mono, channels, memory, create-sound-wave-from-pcm, metasound]
encounters: 1
lastSeen: 2026-09-03T00:10:00Z
---

# There is no route to a mono `USoundWave`

## Gap

Every write path in the plugin produces a 2-channel wave, and nothing converts one:

- **`audio.synth.export`** takes `candidateId`, `name`, `path`, `save`, `overwrite` —
  no channel parameter. The recipe grammar renders to a stereo bus unconditionally
  ("Generators are mono. The mix bus is stereo and each layer's `pan` places it"), so
  a recipe whose layers are all `pan: 0` still exports two identical channels.
  Measured: 17 exports in one session, every response `channels: 2` /
  `payloadChannels: 2`.
- **`audio.authoring.create_sound_wave_from_pcm`** accepts `channels: 1` but its own
  page says the input is upmixed: "Mono is duplicated into both channels, so the
  created wave reports `channels:2` either way."
- **No conversion verb exists.** `texture.channel_extract` / `texture.channel_pack`
  are texture-only; there is no audio analogue, and `set_sound_wave_properties`
  covers `bLooping`, `volume`, `pitch`, `soundGroup`, `compressionQuality`,
  `bMature`, `bSingleLine` — not channel count.

`audio.analysis.audit_folder` already measures the consequence and reports it as a
format fact rather than a defect: over `/Game/FPS/Audio/Waves` (63 waves) it returns
`modalChannels: 2, distinctChannelCounts: 1`.

## Why it matters

On this project all nine MetaSounds declare `UE.OutputFormat.Mono` and read
`Out Mono` from every Wave Player, so the second channel of all 63 waves is decoded
and discarded at playback — 2x the memory and 2x the decode for nothing. That is the
common case for a first-person game: footsteps, impacts, weapon layers and one-shots
are mono sources placed by attenuation, and only the tails and ambience beds want
real stereo. An author who correctly builds a mono graph has no way to feed it mono
assets.

The authored stereo is not merely redundant, it is discarded work: the outdoor and
indoor tails carry a real image (`width` 0.43, `stereoCorrelation` 0.26) that the
mono graph collapses.

## What is wanted

Any one of these closes it; the first is cheapest:

- A `channels: 1` (or `mono: true`) option on `audio.synth.export` that mono-sums the
  stereo bus — or, better, skips the bus and writes the pre-pan layer mix, which is
  already mono by construction, so a mono export is exact rather than a downmix.
- Honour `channels: 1` in `create_sound_wave_from_pcm` instead of upmixing, and fix
  the page that documents the upmix.
- A `audio.authoring.convert_sound_wave_channels {assetPath, channels}` for waves
  that already exist, reporting frames/rate/channels before and after and verifying
  by decode-back the way `export` does.

Whichever ships, `audit_folder` should keep reporting the distribution, and a
`channel_count_outlier` against a folder's mode stays the right check.

## History

- `#1-initial-repro` `OPEN` reporter — Raised from FPS audio review 01 defect 10
  (`Docs/fps/reviews/audio-review-01.md`), which measured all 63 waves at 2 channels
  against nine mono graphs. Investigated whether any supported route to mono exists
  before converting anything: read the `audio.synth.export` parameter list (no
  channel field), the `audio.synth` topology note (stereo bus is not optional),
  `create_sound_wave_from_pcm` (documents the upmix explicitly), and grepped the wiki
  for a conversion verb (only `texture.channel_extract` / `texture.channel_pack`,
  both texture-only). Conclusion: no clean route exists, so nothing was converted —
  filing this rather than hand-rolling a `python.execute` channel rewrite, which
  would leave 63 waves in a state no verb can reproduce or verify.
- `#2-mono-writer-path` `IN-REVIEW` developer — `PwCreateSoundWaveAsset` takes a trailing `NumChannels` (default 2; 1 writes `0.5*(L+R)`, bit-exact for a dual-mono buffer; anything else refused), threaded through `BuildInterleavedClamped` / `BuildInt16SampleBuffer`; the engine writer and the in-place path both take the width off the int16 buffer, and `NumChannels` is already payload-owned so a width change is not a preservation failure. `audio.synth.export` gains `channels: 1|2` (default 2): mono writes the mean of the stereo bus, verifies the decode-back against that mean (`verification.expectedChannels`, per-side RMS/peak), and reports `downmix {method, sourceChannelsIdentical, maxChannelDifference}` so an exact pan-0 mono export is distinguishable from a real fold-down. The ticket's 'better' pre-pan mix was not done: it needs a renderer change in the master chain another agent is editing, and the mean is already exact for the all-`pan: 0` case the ticket describes. `audio.authoring.create_sound_wave_from_pcm` now honours `channels: 1` — **behaviour change**: a mono input creates a 1-channel wave (was duplicated to 2); description fixed. A conversion verb for existing waves was not added (YAGNI; `audio.authoring.set_sound_wave_gain` from `F-rpc-audio-normalize-existing-wave` preserves width, and re-exporting with `channels: 1` covers synth sources). Files: `Source/PinWright/Private/AudioGen/PwAudioExport.{h,cpp}`, `Handlers/Audio/AudioSynthGenerateHandler.cpp` (export only), `Handlers/Audio/SoundWavePcmHandler.cpp`, `Tests/Media/TestSoundWavePcmHandler.cpp`, `docs/wiki-src/audio.synth.md`, `CHANGELOG.md`. Tests: `PinWright.audio.authoring.create_sound_wave_from_pcm.MonoInputWritesMonoWave` (payload header reads 1 channel, one int16 per frame, samples round-trip), `PinWright.audio.authoring.set_sound_wave_gain.MonoWaveStaysMono` (writer-level mono fixture via `PwCreateSoundWaveAsset(...,1)` stays 1-channel through an in-place rewrite).
- `#3-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit d9f345df). run3/full passed non-skipped: `PinWright.audio.authoring.create_sound_wave_from_pcm.MonoInputWritesMonoWave` and `PinWright.audio.authoring.set_sound_wave_gain.MonoWaveStaysMono`. The ticket says any one option closes it. Option 2 is met: `create_sound_wave_from_pcm` now honours `channels: 1`. The payload header reads 1 channel with one int16 per frame, the samples round-trip, and the description was fixed. This is a behaviour change. The writer-level mono path also survives an in-place rewrite. Coverage limit: option 1, `audio.synth.export channels: 1` (mean downmix with a `downmix` report), shipped but no run3 test exercises the export verb's mono path. Neither the exact pre-pan mix nor a conversion verb for existing waves was done. The ticket did not require them.
