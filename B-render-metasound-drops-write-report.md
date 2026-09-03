---
id: B-render-metasound-drops-write-report
title: "audio.synth.render_metasound writes a USoundWave through the shared writer but discards its FPwSoundWaveWriteReport — no routing block, and an in-place rewrite's measured property diff is thrown away"
status: OPEN
severity: Medium
category: bug
tags: [audio, synth, render-metasound, export, soundwave, soundclass, attenuation, verification, updated-in-place, response-contract, consistency]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# `audio.synth.render_metasound` measures the write report and publishes none of it

Follow-up to `B-synth-export-wipes-soundclass-attenuation`, found by source audit
rather than a live run — the two fixes landed concurrently and the seam between them
was never closed.

`B-synth-export-wipes-soundclass-attenuation` changed `PwCreateSoundWaveAsset` to
fill an `FPwSoundWaveWriteReport` (`AudioGen/PwAudioExport.h:76`) and added
`PwAddSoundWaveWriteReport` (`:155`) to publish it as `routing.soundClass` /
`routing.attenuationSettings` on the result and `verification.propertiesPreserved` /
`verification.changedProperties` on the verification.
`F-metasound-no-render-to-pcm` added `audio.synth.render_metasound`, which writes a
`USoundWave` through that same writer when `name`+`path` are given
(`Handlers/Audio/AudioSynthRenderMetaSoundHandler.cpp:522`). Its compile-fix pass
adapted the new signature and stopped there: the report is filled, one field
(`bSavedToDisk`) is read, and the rest is dropped on the floor.

## Caller table

Every caller of `PwCreateSoundWaveAsset` in the plugin:

| Call site | Verb | Reaches the in-place branch? | `routing` published | `propertiesPreserved` / `changedProperties` | `ChangedProperties` consumed at all |
|---|---|---|---|---|---|
| `SoundWavePcmHandler.cpp:286` | `audio.authoring.create_sound_wave_from_pcm` | yes | yes (`:390`) | yes (`:390`), folded into `verification.pass` | yes |
| `AudioSynthGenerateHandler.cpp:1979` | `audio.synth.export` | yes | yes (`:2075`) | yes (`:2075`), folded into `verification.pass` | yes |
| `AudioMusicHandler.cpp:1426` | `audio.music.export_stems` | yes | **no** | no — deliberate, per-row response budget (comment at `:1442`) | yes → per-stem `VERIFICATION_FAILED` |
| `AudioSynthRenderMetaSoundHandler.cpp:522` | `audio.synth.render_metasound` | **yes** | **no** | **no** | **no — computed, then discarded** |

`render_metasound` is the only site that measures the diff and uses none of it.

## The in-place path is reachable, so the missing half is load-bearing

Not hypothetical. `render_metasound` resolves its target at
`AudioSynthRenderMetaSoundHandler.cpp:379`:

```cpp
Resolution = AssetCreatePolicy::Resolve(ValidatedPath, AssetName,
    USoundWave::StaticClass(), /*bOverwriteRequested=*/false,
    /*bRequireExactClass=*/true);
```

`AssetCreatePolicy::Resolve` rule 2 (`Utils/AssetCreatePolicy.h`) is "class matches,
overwrite not requested → `UpdateInPlace`", which the handler's own comment at `:375`
states outright ("the only outcomes are Create, UpdateInPlace and Rejected"). A
matching `USoundWave` at the path then reaches `PwCreateSoundWaveAsset`'s exact-class
branch (`PwAudioExport.cpp:358`), which sets `bUpdatedInPlace`, snapshots every
non-payload `UPROPERTY` and diffs it after the rewrite. So the natural iterate loop —
tweak the MetaSound, re-render over the same wave — takes the exact path whose
preservation guarantee the parent ticket built the measurement for, and the response
carries no trace of it.

It is worse than silence: `AssetCreatePolicy::AddCreateReport` at `:576` publishes
`mode: "updated_in_place"`, so the caller is told an in-place rewrite happened, on a
verb whose wiki says the write goes "through the same writer `audio.synth.export`
uses, behind the same non-modal overwrite gate" (`docs/wiki-src/audio.synth.md:103`) —
and is then handed a `verification` block built from frames and sample rate only
(`:550-556`). The three sibling verbs treat a non-empty diff as a hard failure; this
one answers `pass: true` beside it.

Both halves are gaps, with different urgency:

- **`routing` (present-day).** Every wave `render_metasound` *creates* is unrouted —
  null `SoundClassObject` / `AttenuationSettings` — and the response never says so.
  That is encounter `#4` of the parent ticket verbatim ("`mode:"created"` has the same
  end state as `mode:"updated_in_place"`, for a different reason"), whose stated fix
  was that a create must report the two empty strings rather than say nothing. Three
  verbs do; this one does not. The caller's fallback is an extra `asset.dump` or two
  `property.get` calls per wave, and the four encounters on the parent ticket are the
  record of what happens when nobody thinks to take them.
- **`changedProperties` (latent).** The diff is empty by construction today — the
  in-place path assigns payload fields and nothing else — so no false success can
  actually occur yet. It exists so a future engine field that starts moving under the
  rewrite fails the verb loudly. When that day comes, three verbs fail and
  `render_metasound` silently ships the unrouted asset. The detector is installed and
  unwired on one of four sites.

## Secondary: `audio.music.export_stems` publishes no `routing` either

Same root shape, lower confidence that it is wrong. `AudioMusicHandler.cpp:1438-1443`
deliberately omits the *preservation* block ("the response budget cannot carry a
per-row preservation block that is always true") — a defensible call for a verb that
writes up to 16 rows. But `routing` is not addressed anywhere in that handler, and a
freshly created stem has exactly the null-routing exposure above. A per-row
`soundClass`/`attenuationSettings` pair would cost real budget; an aggregate would not.
Suggested shape rather than the full block: one `unroutedCount` on the result, or a
`soundClass` field emitted per row *only when it is empty*. Worth deciding
explicitly instead of leaving it as an unstated omission.

## Severity reasoning

`Medium` by the rubric's "a readback omits a field and forces a fallback", not `High`:
nothing the verb reports today is a lie — `verification.pass` is scoped to frames and
sample rate and does not claim preservation — and no data is corrupted. Not `Low`
either: the omitted field is the one whose absence produced four logged encounters of
shipped unrouted waves on the parent ticket, and this is not an edge path within the
verb (every `name`+`path` call hits it). The reach modifier argues down — a brand-new
verb is not an every-session method — but the fix is roughly five lines and closes the
last hole in a `High` fix that just landed, so it should not sink below routine
`Medium` work in the picker.

## Fix

1. `AudioSynthRenderMetaSoundHandler.cpp`, in the `bWriteAsset` block: call
   `PwAddSoundWaveWriteReport(AssetBlock, Verification, WriteReport)` and fold the
   diff into the verdict the way the siblings do —
   `pass = bFramesMatch && bRateMatch && WriteReport.ChangedProperties.Num() == 0`,
   with the failure message naming the offending properties.
   **One shape decision to make deliberately, not by default:** this verb namespaces
   its whole asset write under `asset` (`asset.assetPath`, `asset.verification`,
   `asset.save`), whereas the siblings put `routing` at the top level. Passing
   `AssetBlock` yields `asset.routing`, which is internally consistent for this verb
   but not identical to the sibling contract. Pick one, and make
   `docs/wiki-src/audio.synth.md`'s `render_metasound` section say which — it
   currently claims parity with `audio.synth.export`'s write and documents neither
   field.
2. `AudioMusicHandler.cpp`: decide the aggregate-vs-nothing question above and write
   the decision down in the handler comment either way.
3. **Guard against the next caller.** A per-verb automation assertion is the cheap
   half: write over a wave carrying a `USoundClass` and a `USoundAttenuation`, then
   assert the response carries `routing.soundClass` and
   `verification.propertiesPreserved` — mirroring
   `PinWright.audio.synth.export.InPlaceRewriteKeepsNonPayloadProperties`, which the
   parent ticket added. The durable half is a source-scan contract test: every
   `PwCreateSoundWaveAsset(` occurrence under `Private/Handlers/` must be matched by a
   `PwAddSoundWaveWriteReport(` in the same handler body, or by an explicit opt-out
   comment marker for a verb (like `export_stems`) that consumes the report by hand.
   `Tests/Core/TestNoParamHandlersReadNoArgs.cpp` is the working precedent for that
   shape — it resolves the source dir through `IPluginManager` and text-scans handler
   bodies, and documents the same text-heuristic limitation this one would inherit.
   `PwAudioExport.h:138` already argues the report is a required out-parameter "so a
   call site cannot forget it and report persistence, or preservation, it never
   measured"; nothing yet stops a call site from measuring it and then not publishing
   it, which is exactly what happened here.

## History

- `#1-source-audit-after-concurrent-fixes` `OPEN` reporter — Filed from a source audit,
  **not** a live repro: no editor was launched and no RPC was issued, so every claim
  here is code reading with file:line, and the response shapes are read off the
  handlers rather than off captured JSON. Trigger was the observation that
  `B-synth-export-wipes-soundclass-attenuation` and `F-metasound-no-render-to-pcm`
  landed against the same writer in one wave: the first changed
  `PwCreateSoundWaveAsset`'s signature to `FPwSoundWaveWriteReport&`, the second added
  a fourth caller, and the compile-fix pass that reconciled them patched the signature
  without adding the publish. Enumerated all four callers (table above);
  `render_metasound` is the only one that computes `ChangedProperties` and reads
  nothing but `bSavedToDisk` from the report. Verified the in-place branch is
  reachable there — `AssetCreatePolicy::Resolve(..., bOverwriteRequested=false, ...)`
  at `AudioSynthRenderMetaSoundHandler.cpp:379` returns `UpdateInPlace` on a
  same-class occupant by rule 2 of `Utils/AssetCreatePolicy.h`, and
  `PwAudioExport.cpp:358` is the branch that sets `bUpdatedInPlace` — so the
  verification is load-bearing there and not merely a consistency nicety. Also noted
  `audio.music.export_stems` never publishes `routing` (its omission of the
  preservation block is documented and deliberate; the routing omission is unstated),
  recorded as a secondary item rather than a separate ticket because one source-scan
  test would flag both. Dedup: ripgrep over the board for `render_metasound`,
  `write-report` and `routing` found only `F-metasound-no-render-to-pcm` (the verb's
  own feature ticket, `IN-REVIEW`, which does not mention the report) and
  `B-synth-export-wipes-soundclass-attenuation` (the parent, `IN-REVIEW`, whose Fix
  section lists the three verbs it covered and does not name this one).
