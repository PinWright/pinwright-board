---
id: F-rpc-synth-get-recipe
title: "No verb returns a candidate's canonical recipe, so audio.synth.patch's own 'Re-read the recipe' remedy is unfollowable and every pointer must be guessed"
status: OPEN
severity: Medium
category: feature
tags: [audio, synth, patch, canonical-recipe, list-candidates, rfc6902, json-pointer, unactionable-error]
encounters: 2
lastSeen: 2026-09-03T01:30:00Z
---

# The canonical recipe is addressable but not readable

## Symptom

`audio.synth.patch` applies RFC-6902 operations to a candidate's **canonical** recipe, and both
its parameter docs and its Notes tell the caller to read that recipe first:

> Pointer paths address the canonical form, so fetch the recipe (or read a `generate` response)
> rather than guessing: `/layers/0/gainDb`, `/layers/0/generator/params/exciterMs`, ...

A failed `test` op says the same thing:

```
[VERIFICATION_FAILED] JSON Patch operation 0 (test /layers/0/generator/kind) could not be applied:
the recipe's value at '/layers/0/generator/kind' is not the one this op asserted ...
Re-read the recipe rather than editing the patch.
```

**There is no verb that returns it.** `audio.synth.list_candidates` accepts a `fields` allow-list
but has no `recipe` column — `fields: ["id","recipe"]` returns only `{"id": "..."}`, silently
dropping the unknown column rather than erroring. `audio.synth.generate` and `.patch` echo the
render report and analysis but not the recipe. `audio.analysis.to_recipe` fits a *new* draft to
rendered audio; it does not return the recipe a candidate was built from. Nothing else in
`audio.synth` reads.

## Consequence

The "fetch the recipe" instruction is only followable inside the one session and context where
the recipe was authored. For a candidate rendered by an earlier agent, an earlier task, or an
earlier context window, `recipeDigest` (`"6L 2100ms 48000Hz seed71 #7d64fbc1"`) is the whole of
the readable state: layer count, duration, rate, seed, opaque hash. Layer order, generator kinds,
gains, envelopes and effect chains are unreachable, so every pointer past `/durationMs`,
`/seed` and `/master/normalize/*` is a guess, and a guess that lands on the wrong layer renders
silently-wrong audio rather than erroring.

The `test` op does not close the gap: it reports only *that* the assertion failed, never the
actual value, so it cannot be used to probe the document into view. Confirmed by probing
`/layers/0/generator/kind` on `c39_bcf6` with a deliberately bogus value — the error names the
pointer and repeats the un-followable remedy, and discloses nothing about what is there.

## Impact

`Medium`. Nothing is silently wrong when the caller knows the recipe; the cost is that a
resident candidate is only half-usable by anyone who did not author it. In this encounter three
Build 02 candidates were still resident and exactly addressable — `c39_bcf6` (`SW_Explosion_Close`),
`c44_278a` (`SW_Explosion_Distant`), `c61_3ea4` (`SW_Amb_Sea_Loop`) — but only the two whose fix
touched a pointer guessable from the schema (`/master/normalize/{mode,target}` and
`/master/fadeInMs` / `/master/fadeOutMs`) could be repaired by patching. Those two landed as
minimal, surgical edits that preserved every other property of the original design, including
the stereo image. The third needed layer-level edits, so its recipe had to be rewritten from
scratch — discarding a design the reviewer had partly approved, and re-deriving by iteration what
was sitting in the registry the whole time. That is the shape of the cost: the verb pushes
callers from a two-line patch to a full rebuild, precisely when a rebuild is the riskier move.

## Ask

Return the canonical recipe from a read path. Cheapest form: add `recipe` to
`audio.synth.list_candidates`' `fields` allow-list (opt-in, since it is large and would bloat the
default row). A dedicated `audio.synth.get_recipe {candidateId}` is the clearer surface and would
let `patch`'s error text name a real call instead of an instruction with no verb behind it.

Two smaller fixes worth taking either way, independent of the above:

- **Reject unknown `fields` entries.** `fields: ["id","recipe"]` currently returns `{"id": ...}`
  with no warning. Everything else in `audio.synth` rejects an unknown key by design — the
  namespace page calls uniform strictness deliberate — so this one silently-dropped column reads
  as "that candidate has no recipe" rather than "that column does not exist".
- **Stop printing an unfollowable remedy.** Until a read verb exists, `VERIFICATION_FAILED`
  should not end with "Re-read the recipe rather than editing the patch."

## History
- `#1-filed` `OPEN` reporter — Hit while fixing the Build 03 wave defects from
  `Docs/fps/reviews/audio-review-02.md` §5. Needed to reshape the sub layer of
  `SW_Explosion_Close`, whose exported Build 02 candidate `c39_bcf6` was still resident in the
  64-slot registry. `list_candidates` with `fields: ["id","recipe"]` returned `{"id":"c43_53e1"}`;
  a `test` probe on `/layers/0/generator/kind` returned `VERIFICATION_FAILED` naming the pointer
  and disclosing no value. With no way to learn which of the six layers was the sub, the recipe
  was rewritten from scratch and re-converged over four `generate` iterations against the
  analysis and the waveform/spectrogram plots. By contrast `SW_Explosion_Distant` (`c44_278a`)
  and `SW_Amb_Sea_Loop` (`c61_3ea4`) were fixed with one `patch` each, against pointers the
  schema guarantees exist regardless of layer layout — the sea loop's seam regression
  (`startDiscontinuity 0.0072` / `endDiscontinuity 0.0054`) went to exactly `0` / `0` from a
  two-op patch setting `fadeInMs`/`fadeOutMs` to 4, preserving its eight layers, its seed and
  its stereo width (correlation 0.9285) untouched.

- `#2-round-robin-variants-have-no-source-to-vary-from` `OPEN` reporter — Second encounter,
  and it names a use case the `#1` framing does not cover: **authoring a round-robin variant of
  a shipped asset.** `Docs/fps/reviews/audio-review-03.md` §5 item 3 asks for a `_B` sibling of
  `SW_Fire_{AR,Pistol}_{Mech,Body}` that is "the same gun" but not a repeat — which is
  definitionally a small perturbation of the original's recipe. The originals were exported in
  Build 01; their candidates are long gone from the 64-slot registry, and a recipe is not stored
  on the `USoundWave`, so there is nothing to perturb. `audio.synth.variations` — the verb built
  for exactly this job — takes a `candidateId` and therefore cannot touch a shipped asset at all.
  The fallback cost, measured: 4 originals reverse-engineered from `audio.analysis.analyze` peaks
  plus `audio.analysis.decompose` modes, then **21 `generate` iterations** to re-converge four
  variants onto the originals' duration / peak / crest / six-band split. `to_recipe` was not
  usable as the shortcut here — it fits a *draft* to the reference rather than recovering the
  authored recipe, so its mode set and layer count are its own, and a variant built on it is a
  sibling of the draft, not of the shipped wave.
  Two asks, in order of value for this case:
  1. **Persist the recipe on the exported `USoundWave`** (an editor-only string property or an
     asset-registry tag), so `variations` and `patch` can take an `assetPath` the way `analyze`
     and `decompose` already do. That single change turns "author a round robin" from 21
     iterations into one `variations` call.
  2. Failing that, the read verb `#1` asks for, so a resident candidate can at least be
     round-tripped without guessing pointers.
