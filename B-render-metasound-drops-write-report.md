---
id: B-render-metasound-drops-write-report
title: "audio.synth.render_metasound writes a USoundWave through the shared writer but discards its FPwSoundWaveWriteReport — no routing block, and an in-place rewrite's measured property diff is thrown away"
status: DONE
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
- `#2-write-report-published-under-asset` `IN-REVIEW` developer — `audio.synth.render_metasound` now calls `PwAddSoundWaveWriteReport(AssetBlock, Verification, WriteReport)`, so the write report lands under the verb's own `asset` namespace (`asset.routing.soundClass` / `asset.routing.attenuationSettings` on every write, `asset.verification.propertiesPreserved` + `changedProperties` on an in-place rewrite) — chosen deliberately over top-level `routing` because this verb's top level describes the render and every write field already lives under `asset`; `docs/wiki-src/audio.synth.md` render_metasound section says so. The diff is folded into the verdict (`pass = framesMatch && sampleRateMatch && propertiesPreserved`), and a write that does not verify now fails the call with `VERIFICATION_FAILED` (or the decoder's code when it cannot decode) naming the changed properties and carrying the result plus the resident `candidateId` — **behaviour change**: it used to answer success beside `pass:false`. `audio.music.export_stems` decision: publishes aggregate `routing.withoutSoundClass` / `routing.withoutAttenuation` counts (not per-row blocks; attenuation-free stems are often intended for 2D music), with the reason at the call site under a `PW_SOUNDWAVE_WRITE_REPORT_BY_HAND` marker. Guard against the next caller: new source-scan contract test requiring each handler file's `PwCreateSoundWaveAsset(` calls to be matched by `PwAddSoundWaveWriteReport(` calls or that marker. Files: `Source/PinWright/Private/Handlers/Audio/AudioSynthRenderMetaSoundHandler.cpp`, `Handlers/Audio/AudioMusicHandler.cpp`, `Tests/Media/TestPwMetaSoundRender.cpp`, `Tests/Infra/TestSoundWaveWriteReportPublished.cpp` (new), `docs/wiki-src/audio.synth.md`, `CHANGELOG.md`. Tests: `PinWright.audio.synth.render_metasound.InPlaceRewritePublishesWriteReport` (render+write over an in-memory sine MetaSound, assign SoundClass+attenuation, re-render, assert `asset.mode: updated_in_place`, `asset.routing` names both, `propertiesPreserved: true`), `PinWright.infra.contract.SoundWaveWriteReport.EveryWriterPublishesIt` (fails on the pre-fix render handler: 1 write, 0 publishes).
- `#3-review-fixes` `IN-REVIEW` developer — Review fixes. (1) `audio.music.export_stems`' `routing.withoutSoundClass` / `routing.withoutAttenuation` are documented in `docs/wiki-src/audio.music.md`. `PinWright.audio.music.export_stems.VerifiesAndIsIdempotent` asserts 1/1 on the first export; it then routes the stem to a SoundClass and asserts 0/1 on the re-export. (2) Contract scanner `Tests/Infra/TestSoundWaveWriteReportPublished.cpp` counts calls in `NeutralizeSourceText` output, so block comments are blanked too. An unreadable handler file is now `AddError` instead of a silent skip. The header notes that the opt-out marker is counted in raw text. (3) `audio.synth.md` now says a decode failure returns the decoder's code. Known gap: no test drives `render_metasound`'s failing-write path (`VERIFICATION_FAILED` with the resident `candidateId`); only the passing in-place rewrite is tested.
- `#4-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit d9f345df). run3/full passed non-skipped: `PinWright.audio.synth.render_metasound.InPlaceRewritePublishesWriteReport`, `PinWright.infra.contract.SoundWaveWriteReport.EveryWriterPublishesIt` and `PinWright.audio.music.export_stems.VerifiesAndIsIdempotent`. Fix 1: after re-rendering over a routed wave, the response carries `asset.mode: updated_in_place` and `asset.routing` naming both the SoundClass and the attenuation, with `propertiesPreserved: true`. The `asset.` namespace was chosen deliberately and is documented. Fix 2: export_stems publishes aggregate `routing.withoutSoundClass` / `withoutAttenuation`, asserted at 1/1 and then 0/1 after routing. Fix 3: a source-scan contract test requires every `PwCreateSoundWaveAsset(` call to be matched by a publish or by the opt-out marker. Coverage limit: no test drives the failing-write path (`VERIFICATION_FAILED` with `candidateId`). The diff is empty by construction today, as the ticket notes.
