---
id: B-synth-patch-canonical-omits-layer-fx
title: "audio.synth.patch: a layer declared without `fx` has no `fx` array in the CANONICAL recipe, so the documented RFC-6902 append `/layers/N/fx/-` fails with 'no parent container'"
status: OPEN
severity: Medium
category: bug
tags: [audio, synth, patch, canonical-recipe, rfc6902, json-pointer, wiki-mismatch]
encounters: 1
---

# `/layers/N/fx/-` is unusable on a layer that was written without an `fx` key

## Symptom

`audio.synth.patch` promises pointers into a canonical recipe in which every optional value
is explicit. Appending an effect to a layer that had no `fx` in the original recipe is
rejected:

```
[INVALID_PARAMS] JSON Patch operation 1 (add /layers/3/fx/-) could not be applied:
path segment '-' has no parent container; the pointer walks through a value that does not exist.
{"opIndex":1,"op":"add","path":"/layers/3/fx/-","reason":"...","sourceCandidateId":"c76_7642"}
```

No candidate is created, so the whole patch (including its valid `replace` ops) is lost.

## Repro

Candidate `c76_7642`, a 4-layer close-explosion recipe. Layers 0, 1 and 2 were authored with
an `fx` array; layer 3 (the debris `noise` layer) was authored without one. Then:

```json
[{"op":"replace","path":"/layers/3/gainDb","value":6},
 {"op":"add","path":"/layers/3/fx/-","value":{"kind":"filter","params":{"type":"lowpass","cutoffHz":3500,"resonance":0.8}}}]
```

fails as above. The same patch with the array supplied whole succeeds and renders normally:

```json
[{"op":"replace","path":"/layers/3/gainDb","value":6},
 {"op":"add","path":"/layers/3/fx","value":[{"kind":"filter","params":{"type":"lowpass","cutoffHz":3500,"resonance":0.8}}]}]
```

So `/layers/3/fx` is genuinely **absent** from the canonical form rather than present-and-empty;
the `add` created it.

## What the documentation promises

`audio.synth.patch` parameter docs:

> Paths are RFC-6901 pointers into the CANONICAL recipe, in which every optional value has been
> made explicit.

Its Notes repeat it and give the reader the exact mental model that fails here:

> The patch applies to a fresh serialization of the source candidate's CANONICAL recipe - every
> optional value made explicit ... Pointer paths address the canonical form, so fetch the recipe
> (or read a `generate` response) rather than guessing: `/layers/0/gainDb`,
> `/layers/0/generator/params/exciterMs` ...

`audio.synth.cookbook` makes the same claim for the round-trip:

> That is what makes a recipe round-trip: parse then serialize returns the same recipe with
> defaults made explicit, so patching serialized output is safe.

`exciterMs` is indeed materialised when never written, which is what makes the omission of an
empty `fx` array surprising rather than merely undocumented — the canonicalisation is applied to
scalar defaults but not to the optional array containers (`fx` at minimum; `ampEnvelope`,
`pitchEnvelope` and `modulation` are likely the same shape and were not tested).

## Impact

`Medium`. Correct output, clear error naming the exact pointer, and a one-line workaround, so
nothing is silently wrong. The cost is that the natural incremental-authoring move — "render the
layers, listen, then add a filter to the layer that needs one" — fails on precisely the layers
most likely to need it (the ones deliberately authored bare), and it takes the valid operations
in the same patch down with it. It also makes the canonicalisation guarantee unreliable in
general: an author who trusts the sentence cannot know which optional values it covers without
testing each one.

**Workaround:** never append to a layer effect chain with `/-`. Supply the whole array with
`add /layers/N/fx` (safe when the layer has no chain, destructive when it does), or `replace`
the full array when the layer already has one.

**Fix:** emit `"fx": []` for every layer during canonical serialization (and the equivalent for
any other optional container), so `/-` appends work uniformly. Failing that, correct the three
documentation sentences above to say that optional *arrays* are omitted when empty while scalar
defaults are materialised, and name the containers affected.

## History
- `#1-filed` `OPEN` reporter — Hit while synthesizing 20 bullet-impact and explosion waves into `/Game/FPS/Audio/Waves/Impacts/`. Adding a lowpass to the debris layer of a close-explosion recipe via `add /layers/3/fx/-` returned `INVALID_PARAMS` "path segment '-' has no parent container; the pointer walks through a value that does not exist", discarding the `replace /layers/3/gainDb` operation batched with it; layers 0-2 of the same recipe carried `fx` arrays and would have accepted the identical append. Re-issuing as `add /layers/3/fx` with the array supplied whole succeeded and produced candidate `c93_e453`, proving the key was absent rather than empty. Contradicts `audio.synth.patch` ("pointers into the CANONICAL recipe, in which every optional value has been made explicit"), its Notes, and `audio.synth.cookbook` ("parse then serialize returns the same recipe with defaults made explicit, so patching serialized output is safe") — scalar defaults such as `generator/params/exciterMs` are materialised, but the optional `fx` array container is not. Rated Medium: the error is loud and specific and the workaround is one line, but it breaks the incremental "add an effect to the layer that needs it" loop and silently costs the other operations batched into the same patch.
