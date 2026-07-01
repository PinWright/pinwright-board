---
id: F-sound-wave-property-edit
title: "No way to edit USoundWave properties (loop, volume, pitch, sound group, compression) after import"
status: DONE
severity: Low
category: feature
tags: [audio, sound-wave, authoring, properties, asset-edit]
---

# No way to edit USoundWave properties after import

The `audio.authoring` namespace can author USoundCue, USoundClass,
USoundMix, and USoundSubmix assets and tweak their per-asset scalars,
but it has no equivalent verb for USoundWave property edits. After a
WAV is imported, every wave-level field that ships on
`USoundWave` — `bLooping`, `Volume`, `Pitch`, `SoundGroup`,
`CompressionQuality`, `bMature`, `bSingleLine`, the modulation
defaults — is unreachable from the RPC surface.

Verified gap, against
`Source/EditorAutomationRpcGateway/Private/Handlers/Audio/`:

1. **No write-side RPC for USoundWave fields.** Grep for
   `REGISTER_RPC_HANDLER.*sound_wave` and
   `REGISTER_RPC_HANDLER.*SoundWave` across `Handlers/` returns zero
   hits. `LoadSoundWaveFromPath` (AudioAuthoringHandler.cpp:206) is
   used only to resolve a wave path into a `USoundWave*` for cue
   wiring or for the dialogue-context mapping table — never to mutate
   the wave's own properties. The dialogue / cue handlers consume
   waves; nothing in the namespace produces or edits one.

2. **Readback exists, write doesn't.** `SoundWaveDumpBuilder.cpp`
   already serialises `duration`, `numChannels`, `sampleRate`,
   `bLooping`, `soundGroup`, `volume`, `pitch` — confirming these are
   the canonical per-wave fields the plugin already recognises. The
   read/write asymmetry means asset-dump shows the current value but
   no RPC can change it.

3. **Import side is already covered.** `asset.import`
   (AssetManageHandler.cpp:84) explicitly supports WAV via the
   automated UFactory pipeline ("FBX, PNG, WAV, etc. — picked by
   extension"). So the *import* half of the original proposal is
   redundant; this ticket is scoped to the *property-edit* half only.

**Impact:** End-to-end headless audio bring-up still needs a manual
editor session to flip `bLooping` on a one-shot loop, set
`SoundGroup` to `Voice` on dialogue waves, or override
`CompressionQuality` for music stems. Most projects route through a
USoundCue and edit cue-level settings (which *is* covered by
`audio.authoring.set_cue_properties`), which is why this is Low: it
only bites projects that play USoundWaves directly or that bake
per-wave defaults expected to propagate through every cue that
references the wave.

**Fix:** Add `audio.authoring.set_sound_wave_properties(assetPath,
{bLooping?, volume?, pitch?, soundGroup?, compressionQuality?,
bMature?, bSingleLine?}, save?)` mirroring the existing
`set_class_properties` / `set_cue_properties` shape:

- Load the wave via `LoadSoundWaveFromPath` (reuse the existing
  helper).
- Apply each field only when present in params (existing per-field
  conditional pattern in the handler).
- `SoundGroup` accepts the `ESoundGroup` enum name string
  (`SOUNDGROUP_Default`, `SOUNDGROUP_Voice`, `SOUNDGROUP_Effects`,
  ...); reuse `StaticEnum<ESoundGroup>()` already imported in the
  dump builder.
- `CompressionQuality` is `int32` (0–100, UE convention).
- Call `Modify()` + `MarkPackageDirty()` + optional save (match the
  rest of the namespace).
- Return the post-edit field values for verification, matching the
  set of fields `SoundWaveDumpBuilder` already exposes.

Out of scope: editing `Duration`, `NumChannels`, `SampleRate` —
those are content-derived and only meaningful as readback.

Tests under `Source/EditorAutomationRpcGatewayTests/`: import a
fixture WAV (or use an existing one in the test content), flip
`bLooping`, set `SoundGroup=Voice`, set
`compressionQuality=50`, re-dump via `asset.dump`, assert all three
round-trip.

Broad UE audience: any project that ships per-wave SoundGroup splits
(dialogue/SFX/music) or per-wave compression tuning needs this; the
manual workaround (open each wave, edit, save) does not scale past
~dozen waves.

## History
- `#1-initial-scope` `OPEN` reporter — Proposed combined import+edit RPC pair; on verification, `asset.import` (AssetManageHandler.cpp:84) already routes WAV through UFactory, so the import half is redundant. Scoped this ticket to the property-edit half only. Confirmed no existing `set_sound_wave_properties` (zero hits for `REGISTER_RPC_HANDLER.*[Ss]ound_?[Ww]ave` in `Handlers/`), confirmed the writable field set against the existing `SoundWaveDumpBuilder` readback (`bLooping`, `Volume`, `Pitch`, `SoundGroup`), and noted modulation defaults / `CompressionQuality` / `bMature` / `bSingleLine` as the natural additions. No prior audio-board ticket overlaps — existing F-audio-* entries cover Submix routing, MetaSound, and SoundCue, not USoundWave property edits.
- `#2-implemented-set-sound-wave-properties` `IN-REVIEW` developer — Added `audio.authoring.set_sound_wave_properties` in new file `Handlers/Audio/SoundWaveAuthoringHandler.cpp` (sibling to existing AudioAuthoringHandler.cpp — kept separate to avoid touching the in-flight diff there). Inlined a small wave loader (≈6 lines) instead of lifting the existing LoadSoundWaveFromPath helper. Supports bLooping, volume, pitch, soundGroup (ESoundGroup name string round-trips with SoundWaveDumpBuilder), compressionQuality (int32), bMature, bSingleLine, with optional save flag. Returns post-edit field echo + AddAssetVerification. Tests in `Tests/Media/TestSoundWaveAuthoringHandler.cpp` cover full round-trip + INVALID_SOUND_GROUP + SOUND_WAVE_NOT_FOUND counterfactuals. No Build.cs change.
- `#3-verify-fix` `DONE` tester — Verified: schema `audio.authoring.set_sound_wave_properties?` exposes all proposed fields (bLooping, volume, pitch, soundGroup, compressionQuality, bMature, bSingleLine, save). Live call on `/Game/Audio/FPV_SOUND/UI/UISFX_Select_7` (save=false) with bLooping=true/volume=0.75/pitch=1.1/soundGroup=SOUNDGROUP_Voice/compressionQuality=50 round-tripped all values in the response echo. Counterfactual call with soundGroup="NOT_A_REAL_GROUP" returned typed error `INVALID_SOUND_GROUP` as designed.
