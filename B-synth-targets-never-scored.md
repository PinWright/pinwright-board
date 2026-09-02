---
id: B-synth-targets-never-scored
title: "audio.synth recipe `targets` are parsed, validated and round-tripped but NEVER scored — a missed target is silently indistinguishable from a met one"
status: OPEN
severity: High
category: bug
tags: [audio, synth, generate, targets, silent-false-success, contract, wiki-mismatch]
encounters: 1
lastSeen: 2026-09-02T19:15:00Z
---

# `targets` is accepted and then silently ignored by `audio.synth.generate`

## Symptom

A recipe's `targets` block is fully honoured by the parser — unknown keys inside it are
rejected, `"at least one of min / max on every targets entry"` is enforced, and the block
survives the parse/serialize round trip — but `audio.synth.generate` returns **no scoring of
any kind**. There is no `targets`, `score`, `report` or `misses` key in the response, and a
target the render plainly missed produces no line anywhere.

Because absence of a report line is the *only* signal a caller has, a missed target and a met
target are byte-identical in the response. The natural reading of a clean response is "every
target was met", which is a lie whenever one was not.

## Repro

Both of these were real calls in one session (UE 5.8, EAContentExamples58, gateway 27145).

**1.** `audio.synth.generate` with a `targets` block containing
`"centroidHz": { "min": 900, "max": 3500 }` (plus `peakDb`, `attackMs`, `crestDb`,
`clippedSamples` entries). The response's own `analysis.spectral.centroidHz` came back
**4843.5** — 1343 Hz above the stated maximum. No miss was reported.

**2.** `audio.synth.generate` with `"peakDb": { "min": -1.5, "max": -0.5 }`. The response's
own `analysis.technical.peakDb` came back **-2.47**, about 2 dB below the stated minimum. Again
no miss was reported.

The complete key set of both responses was:
`candidateId, reused, frames, sampleRate, channels, durationSeconds, render, analysis, images,
registry, message`. Nothing target-shaped in it.

Note that in both cases the handler had already computed the exact metric it would need to
compare against — the miss is visible in the same response object, one key away from the target
the caller wrote. The caller has to re-implement the comparison by hand for every metric, which
is precisely the work the `targets` block exists to remove.

## What the documentation promises

`audio.synth.cookbook` states it plainly:

> `targets` never changes the render. It is scored against the analysis afterwards, so a miss is
> a report line rather than an error, and a metric the signal cannot express (a decay time on a
> sustained tone) is scored as **unscored** rather than as a failure against zero.

`audio.synth.describe_schema` reinforces it: `targets` is a documented top-level recipe member,
it has a whole `section: "targets"` drill-down describing "the metric names targets accepts", it
appears in the schema's own `example`, and `required` includes an entry governing its shape. The
`audio.synth` namespace page describes the loop as "iterated on against measured numbers".

Every one of those pages describes a scoring step that does not exist. Three distinct
"unscored / scored / miss" behaviours are documented in that cookbook sentence alone, and none
of the three can be observed.

## Root cause

Read from source, not inferred. `targets` is parsed at
`Plugins/PinWright/Source/PinWright/Private/AudioGen/PwSynthRecipe.cpp:1623-1628`, is a
first-class recipe member in the allowed-key list at `:1711`, lands in
`FPwSynthRecipe::Targets` (`PwSynthRecipe.h:387`, a `TArray<FPwSynthTargetRange>`), and is
written back out at `:1987` — which is what makes the round trip clean.

But `Handlers/Audio/AudioSynthGenerateHandler.cpp` never reads `Targets` at all. Its only two
matches for the string are at `:1431` and `:1552`, and both belong to the `variations`
verb's `mutations` array ("'mutations' holds %d targets"), unrelated to recipe targets. A
repo-wide grep for a scoring symbol (`ScoreTargets`, `targetScore`, `targetReport`) finds
nothing. The field is stored and echoed; no code ever compares it to the analysis.

That also explains why this has stayed invisible: the round trip works, `describe_schema`
documents it, the parser rejects malformed entries, and `audio.synth.patch` can patch the block
— every surface *around* the feature behaves, so the recipe author has no reason to suspect the
comparison itself is absent.

## Impact

`High`, per the board rubric's "silent false-success ... the caller trusts a result that is a
lie and builds on it". The reach modifier pushes it up rather than down: `targets` is in the
schema's own worked example and in every recipe in `audio.synth.cookbook`, which is the
documented cold-start path for a namespace that ships no preset library, so essentially every
synth session sets targets and every one of them is silently unscored.

Not rated Critical only because nothing is corrupted or lost — the numbers needed to catch the
miss are present in the same response, so a caller who distrusts the feature can recover.

**Workaround:** ignore `targets` entirely as an output contract. Treat it as inert
documentation-in-the-recipe, and compare `analysis.technical.*`, `analysis.envelope.*` and
`analysis.spectral.*` against the intended ranges by hand in the caller after every
`generate`. That is what this session did for all weapon-layer renders.

**Fix:** either implement the scoring the three wiki pages already promise — emit a `targets`
block in the `generate` response with one row per entry carrying `metric`, `min`, `max`,
`measured`, and a status of `met` / `missed` / `unscored`, with `unscored` used when the
analysis omitted that metric family (the `audio.analysis` "absence is a measurement" rule makes
this natural) — or, if scoring is not going to be implemented, reject `targets` at parse time
with a typed error and delete the promise from `audio.synth.cookbook`,
`audio.synth.describe_schema` and the `audio.synth` namespace page. The present state is the
worst of the three, because it reads as a working feature from every direction except the one
that matters.

## History
- `#1-filed` `OPEN` reporter — Hit while designing layered weapon fire for the FPS AUDIO stream on EAContentExamples58 (UE 5.8, shared editor, port 27145). Two consecutive `audio.synth.generate` calls carried `targets` blocks and both missed a stated bound by a wide margin — `centroidHz` measured 4843.5 against a `{min:900,max:3500}` target, then `peakDb` measured -2.47 against a `{min:-1.5,max:-0.5}` target — and neither response carried any scoring key at all; the full key set both times was `candidateId, reused, frames, sampleRate, channels, durationSeconds, render, analysis, images, registry, message`. Confirmed in source rather than guessed: `targets` is parsed (`AudioGen/PwSynthRecipe.cpp:1623-1628`), is an allowed top-level key (`:1711`), lands in `FPwSynthRecipe::Targets` (`PwSynthRecipe.h:387`) and is serialized back (`:1987`), but `Handlers/Audio/AudioSynthGenerateHandler.cpp` never reads the field — its only `Targets` matches (`:1431`, `:1552`) are the unrelated `variations` `mutations` cap — and no scoring symbol exists anywhere in the module. The misleading pages are `audio.synth.cookbook` ("scored against the analysis afterwards, so a miss is a report line ... scored as unscored rather than as a failure against zero"), `audio.synth.describe_schema` (which ships a whole `section: "targets"` drill-down plus a `targets` block in its worked example), and the `audio.synth` namespace page ("iterated on against measured numbers"). Rated High for silent false-success with an every-session reach: a clean response is indistinguishable from a met target, so the documented iterate-until-in-range loop silently never terminates on evidence. Workaround adopted for the rest of the stream: treat `targets` as inert and hand-compare the `analysis.*` scalars in the caller after every render.
