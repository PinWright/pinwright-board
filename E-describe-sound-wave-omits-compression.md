---
id: E-describe-sound-wave-omits-compression
title: "describe_sound_wave wiki promises 'compression settings' and recommends it 'after compression-setting edits', but the output omits compressionQuality entirely"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, audio, sound-wave, describe, compression-quality, readback, round-trip, wiki]
---

# `describe_sound_wave` doc claims compression settings it never returns

`audio.authoring.set_sound_wave_properties` accepts and writes
`compressionQuality` (int32 0..100), and its response echoes the value back.
But the documented readback for SoundWaves, `audio.authoring.describe_sound_wave`,
**omits `compressionQuality` from its output entirely** — even though its own
wiki explicitly promises "compression settings" and recommends the tool
specifically for verifying compression edits. So the natural
set-then-describe-to-confirm round-trip the doc tells you to run cannot actually
confirm the one editable compression field; the value can only be read back off
the `set_*` echo, never via the recommended reader.

This is the docs/ergonomic half of the SoundWave compression readback gap. The
underlying field omission overlaps the read-side parity work in
`E-dump-rpc-parity` (sound_wave row — there scoped to whether a describe RPC
*exists*, now closed) and the writer in `F-sound-wave-property-edit` (DONE —
which deliberately added `compressionQuality`/`bMature`/`bSingleLine` writers
*beyond* the field set `SoundWaveDumpBuilder` reads back). Neither flags the
distinct, sharper problem here: the `describe_sound_wave` **documentation
actively contradicts its own output** and steers callers to it for exactly the
verification it cannot perform.

## The contradiction (quotable)

The `describe_sound_wave` wiki (`docs/wiki-src/audio.authoring.md`) makes two
explicit compression claims:

1. Namespace overview line:
   > "... and `describe_sound_wave` for raw SoundWave **compression** / duration / channel metadata."

2. The method's own Notes block:
   > "Return the live `sound_wave.json` payload for a `USoundWave`: format /
   > sample rate / channels / duration / loop flag / **compression settings**.
   > Delegates to the same `SoundWaveDumpBuilder` used by `asset.dump` ... Use
   > this for raw-wave inspection after import or **compression-setting edits**,
   > when writing a full dump folder is overkill."

But `SoundWaveDumpBuilder::BuildSoundWaveJson`
(`Source/EditorAutomationRpcGateway/Private/Handlers/Asset/SoundWaveDumpBuilder.cpp:20-50`)
emits only `duration`, `numChannels`, `sampleRate`, `bLooping`, `soundGroup`,
`volume`, `pitch` — and **never sets a `compressionQuality` field** (nor any
other compression field). The doc's "compression settings" and
"after compression-setting edits" recommendation describe output that does not
exist.

## Verbatim repro (replay-confirmed via mcp__editor-automation__call)

Asset: `/Game/ExampleContent/Audio/Audio/Surfaces/Audio_Footstep_Wood01`

Write compressionQuality and observe the writer echoes it:
```
call("audio.authoring.set_sound_wave_properties",
     { assetPath:".../Audio_Footstep_Wood01", compressionQuality:80, save:true })
-> {"bLooping":false,"volume":1.25,"pitch":0.8999999761581421,
    "compressionQuality":80,"bMature":false,"bSingleLine":false,
    "soundGroup":"SOUNDGROUP_Effects","message":"SoundWave properties updated", ...}
```

Now read it back with the documented reader:
```
call("audio.authoring.describe_sound_wave", { assetPath:".../Audio_Footstep_Wood01" })
-> {"duration":0.6976666450500488,"numChannels":1,"sampleRate":48000,
    "bLooping":false,"soundGroup":"SOUNDGROUP_Effects",
    "volume":1.25,"pitch":0.8999999761581421}
```

No `compressionQuality` key. `get_audio_info` is even thinner (only
`duration`/`sampleRate`/`numChannels`/`assetClass`/`type`), so neither
documented reader confirms a compression edit. The only way to verify
`compressionQuality` today is the `set_*` response echo at write time — which
defeats the "edit, then describe to confirm" workflow the doc prescribes.

## Impact

Low. The write works and the value persists; only the *documented verification
path* is broken. A sound designer following the doc ("inspect after
compression-setting edits with `describe_sound_wave`") gets a response that
silently lacks the field, and must either trust the `set_*` echo or pivot to
`asset.dump` → `properties.json` to confirm. The friction is the misleading doc
+ the silent field omission steering an agent toward a useless confirm call.

## What it should do / how to fix

Pick one — make doc and output agree:

- **Preferred:** add `compressionQuality` (and, for full parity with the
  writer, optionally `bMature`/`bSingleLine`) to
  `SoundWaveDumpBuilder::BuildSoundWaveJson` so describe (and the `sound_wave.json`
  sidecar) actually carry the compression setting the doc promises and the
  writer can set. This closes the set/describe round-trip and honors the wiki
  text as-written. The value is `Wave->GetCompressionQuality()` /
  `Wave->CompressionQuality` (int32).
- **Or (doc-only):** drop the "compression settings" / "compression / duration /
  channel" claims and the "after ... compression-setting edits" recommendation
  from `docs/wiki-src/audio.authoring.md` for `describe_sound_wave`, and state
  that `compressionQuality` is confirmable only via the `set_sound_wave_properties`
  response echo (or `asset.dump` → `properties.json`).

The first is better: it removes the read/write asymmetry `F-sound-wave-property-edit`
introduced (writer added fields the dump builder doesn't read) instead of just
papering over it in docs.

**Workaround:** confirm `compressionQuality` from the `set_sound_wave_properties`
response echo, or via `asset.dump { assetPath }` → `properties.json`; not
`describe_sound_wave`.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed against `/Game/ExampleContent/Audio/Audio/Surfaces/Audio_Footstep_Wood01` via mcp__editor-automation__call: `set_sound_wave_properties { compressionQuality:80 }` echoes `compressionQuality:80` in its response, but `describe_sound_wave` returns only `{duration,numChannels,sampleRate,bLooping,soundGroup,volume,pitch}` — no `compressionQuality` key (and `get_audio_info` returns even fewer fields). Confirmed at source: `SoundWaveDumpBuilder::BuildSoundWaveJson` (SoundWaveDumpBuilder.cpp:20-50) emits no compression field. The `describe_sound_wave` wiki (`docs/wiki-src/audio.authoring.md` lines 7 + 70) explicitly claims it returns "compression settings" / "compression / duration / channel metadata" and recommends it "for raw-wave inspection after ... compression-setting edits" — the doc contradicts the actual output and steers callers to a confirm call that drops the field. Distinct from `F-sound-wave-property-edit` (DONE; added the writer, including compressionQuality, beyond the dump builder's read set) and `E-dump-rpc-parity` (DONE; about describe RPC *existence*, sound_wave row closed) — neither addresses the doc-vs-output mismatch or the broken set/describe round-trip for compressionQuality. Seed method `audio.authoring.set_sound_wave_properties` (works correctly, echoes the value); culprit is the reader `audio.authoring.describe_sound_wave` (+ its wiki overlay).
- `#2-fix` `IN-REVIEW` developer — Took the preferred (root-cause) path: added `compressionQuality` to the one shared builder so describe_sound_wave AND the `sound_wave.json` dump sidecar now carry the field the wiki promises and the writer sets, closing the set→describe round-trip. Changes: (1) `SoundWaveDumpBuilder::BuildSoundWaveJson` (Private/Handlers/Asset/SoundWaveDumpBuilder.cpp) now emits `compressionQuality` via `Wave->GetCompressionQuality()` — the getter, not the bare field, since UE 5.6 made `USoundWave::CompressionQuality` private (same accessor the writer's echo uses at SoundWaveAuthoringHandler.cpp:121); (2) bumped the `sound_wave` dump aspect to version 2 in `AssetDumpCache.cpp`'s `Versions` table (was implicitly at the default 1) so stale `sound_wave.json` caches regenerate with the new field. Regression test: extended `Tests/Media/TestSoundWaveDescribeHandler.cpp` — it now writes `CompressionQuality=73` via the UPROPERTY before invoking `describe_sound_wave`, then asserts the response both has a `compressionQuality` number field and round-trips the exact value (73); this fails if the builder drops the field again. The wiki overlay (`docs/wiki-src/audio.authoring.md`) was left as-is — its "compression settings" / "after compression-setting edits" promises are now honored by the output rather than papered over. Not compiled/tested here; a later phase verifies green.
