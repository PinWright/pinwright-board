---
id: B-soundwave-rewrite-stale-loudness-metadata
title: "SoundWave in-place rewrites leave the 5.8 LUFS / SamplePeakDB metadata describing the old audio"
status: OPEN
severity: Medium
category: bug
tags: [audio, soundwave, loudness, lufs, metadata, asset-registry, 5.8]
encounters: 1
lastSeen: 2026-10-02T00:00:00Z
rice: [1, 2, 1, 1]
priority: 17
---

# In-place SoundWave rewrites leave stale loudness metadata

## Symptom

UE 5.8 added two editor-only, `AssetRegistrySearchable` UPROPERTYs to `USoundWave`: `LUFS` and `SamplePeakDB` (`SoundWave.h:803-809`). The importer fills them through `SetLoudnessValues` (WaveformEditorModule.cpp:27-54), and the `ResaveSoundWaveLoudness` commandlet fills them for assets imported before 5.8. Neither is a payload field in `PwAudioExport.cpp`'s `PayloadOwnedProperties`, so the shared in-place rewrite (`UpdateSoundWaveInPlace`) leaves them unchanged. The non-payload diff then reports them as preserved.

The most visible case is `audio.authoring.set_sound_wave_gain`. A -6 dB rescale leaves `LUFS` / `SamplePeakDB` at the pre-gain values, so the Details panel and any asset-registry query on loudness report the old level. The same applies to every in-place rewrite: `create_sound_wave_from_pcm`, `audio.synth.export`, `render_metasound` and `audio.music.export_stems`. Found while reviewing G15; not reproduced live.

## Expected

After a rewrite, either recompute both values from the new payload (only on engines that have them, 5.8+), or reset them to 0, which the WaveformEditor treats as "not analysed". The response should state which was done.

## History

- `#1-filed-from-g15-review` `OPEN` reporter — Found by reading engine source during the G15 review fixes (`set_sound_wave_gain` cue-point preservation). The engine fields, setter and commandlet are cited above. No live repro.
