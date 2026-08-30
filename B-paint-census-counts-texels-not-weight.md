---
id: B-paint-census-counts-texels-not-weight
title: "landscape.create_procedural_terrain's layer census tallies Weights[...] != 0, so a sibling renormalised from 255 down to 1 contributes 0 to both otherLayerTexelsLost and otherLayerTexelsLostInRegion — on a landscape whose layers really are FinalWeightBlending the exact cost the verb's contract describes is unmeasurable by the instrument sold as measuring it"
status: OPEN
severity: Medium
category: bug
tags: [landscape, create_procedural_terrain, weightmap, layer-paint, census-blind, verify, otherLayerTexelsLost, magnitude, metric-precision, field-naming, latent]
encounters: 1
lastSeen: 2026-08-30T16:55:00+03:00
---

# The census asks "does this texel have any weight", and reports the answer under a name that means "how much weight was lost"

`landscape.create_procedural_terrain`'s `verify` census exists to turn the sibling cost of a layer
paint into a number. Its only observable is a per-texel **presence** test:

```cpp
// Tallies one layer's texels carrying any weight over the FULL extent, split by
// whether they fall inside the painted region.
auto SampleLayerWeights = [&](ULandscapeLayerInfoObject* Info, int32& OutTotal, int32& OutOutside) {
    ...
    CensusRead.GetWeightDataFast(Info, FullMinX, FullMinY, FullMaxX, FullMaxY, Weights.GetData(), 0);
    for (...) for (...) {
        if (Weights[X + Y * SizeX] == 0) { continue; }        // :2908
        ++OutTotal;
        ...
    }
};
// Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp:2893-2917
```

Both published counters are differences of those presence counts:

```cpp
OtherLayerTexelsLost += FMath::Max(0, Entry.OutsideBefore - Entry.OutsideAfter);                 // :3074
OtherLayerTexelsLostInRegion += FMath::Max(0,
    (Entry.TotalBefore - Entry.OutsideBefore) - (Entry.TotalAfter - Entry.OutsideAfter));        // :3075
```

published at `:3139` and `:3140`.

**A sibling renormalised from 255 to 1 is still non-zero, so it is still counted, so both counters
move by zero.** 99.6% of that layer's weight can leave and the instrument reports a clean sheet.
Only a drop all the way to byte 0 is visible at all, and then only as "one texel", identically to a
drop from 2 to 0.

## Why that is a defect and not merely a coarse metric

The magnitude is not incidental to this verb — it is the exact quantity its own contract is about.
The registered summary sells the census as the reason a caller need not go look:

> COSTS THE SIBLING LAYERS: weight-blended layers normalize as a group, so painting one at strength
> 1.0 **drives every other layer in that blend group toward zero** WITHIN the painted region ... so
> the cost to the other layers is a reported number instead of a discovery in Landscape Ed Mode
> (`LandscapeHandler.cpp:2463`)

*"Toward zero"* is a magnitude claim. `otherLayerTexelsLostInRegion` is presented as its
measurement (`Docs/wiki-src/landscape.md:246`: *"Weight the other layers lost **inside** the
region"*). It cannot measure it. On a landscape whose layers really are `FinalWeightBlending`, a
strength-1.0 paint would renormalise every sibling in the region toward — but, per texel arithmetic,
usually not *to* — zero, and **both counters would read 0 through the whole thing**.

The field names compound it. `otherLayerTexelsLost` reads as "weight lost"; the wiki table says
"Weight the other layers lost". The unit is texels-that-went-from-some-weight-to-none. A caller
comparing `otherLayerTexelsLost: 0` against the summary's promise concludes the normalisation did
not happen, or that nothing was lost. Neither follows.

## Re-derived at plugin HEAD `1a9e5778`

`Handlers/Environment/LandscapeHandler.cpp` is clean against `HEAD`
(`git status --porcelain` empty), and every line above was opened for this filing:

- `:2893` — `SampleLayerWeights` declaration, with the comment that names its own unit
  (*"texels carrying any weight"*).
- `:2904-2905` — the single `GetWeightDataFast` read it is built on.
- `:2908` — `if (Weights[X + Y * SizeX] == 0) { continue; }`, the entire discrimination.
- `:2911-2916` — the inside/outside split, applied to the same boolean.
- `:2927-2930` — the census set: `LandscapeInfo->Layers` minus null layer infos and the visibility
  layer.
- `:3074`, `:3075` — the two accumulations; `:3082-3088` — the warning, raised only when
  `OtherLayerTexelsLost > 0`.
- `:3139`, `:3140` — publication.

The painted layer's own readback has the same shape: `texelsWithWeight` counts `Sample != 0`, and
only `texelsAtRequestedWeight` carries any magnitude information — for one layer, at one value.

## Currently latent on this project, and why that is recorded rather than used to downgrade

Nothing on this landscape is weight-blended, so no renormalisation happens and there is no
magnitude change for the census to miss. That is the finding in
`B-paint-auto-created-layer-never-weight-blended`: every `ULandscapeLayerInfoObject` this verb
auto-creates lands on `ELandscapeTargetLayerBlendMethod::None`
(`LandscapeLayerInfoObject.cpp:27` -> `LandscapeSettings.h:167`, unset in this project's `Config/`
and in the engine's), and the merge's final weight-blending pass is gated on `FinalWeightBlending`
(`LandscapeEditLayers.cpp:791`, gate at `:3594`).

So today, on the default path, `otherLayerTexelsLostInRegion: 0` is not merely unrevealing — it is
correct, because the loss it would hide does not occur. **The moment
`B-paint-auto-created-layer-never-weight-blended` lands, real renormalisation begins and this
counter starts certifying clean over it.** That is a dependency worth recording and it is recorded
here.

## Not deferred, and the reason is a measurement, not a judgement

`blockedBy: [B-paint-auto-created-layer-never-weight-blended]` was considered and **rejected**. The
README requires a defer to name a *real* gate, and there is none: a weight-blended landscape is
reachable through PinWright **today**, without any fix. `material.authoring.add_landscape_layer`
with `noWeightBlend: false` explicitly passed calls
`SetBlendMethod(ELandscapeTargetLayerBlendMethod::FinalWeightBlending, false)`
(`MaterialAuthoringHandler.cpp:3211`) — the one code path in the whole plugin that sets a blend
method — so a fixer can mint the fixture, paint two layers over one region, and watch both counters
read 0 while the weights change. A test for this is writable and runnable now. Deferring it would
park a landable ticket behind a gate that does not exist.

## Fix

The read is already there; only the tally is boolean. Accumulate the byte instead of a flag, and
publish the magnitude terms **alongside** the existing counts rather than replacing them — the
texel counts answer a different, still-useful question (how much of the layer is present at all),
and `B-paint-blanks-unallocated-layers-component-wide` and `B-paint-erases-orphaned-layer` both
reason about them.

1. In `SampleLayerWeights`, sum `Weights[...]` into an `OutWeightTotal` / `OutWeightOutside` pair
   next to the existing counts. Zero extra reads — same loop, same buffer.
2. Publish `otherLayerWeightLost` and `otherLayerWeightLostInRegion` (sum of per-layer
   `max(0, before - after)` over the 0..255 byte scale), plus per-layer `weightBefore`/`weightAfter`
   in `layersAffected[]`. A caller can then see 255 -> 1 as the 254-per-texel loss it is.
3. Raise the outside-region warning on the **weight** term as well, not only the texel term
   (`:3082`). A sibling reduced but not zeroed outside the region is exactly as much a defect as
   one zeroed, and today only the second is warned about.
4. Correct the two naming sites that invite the over-read: the wiki table's *"Weight the other
   layers lost"* (`Docs/wiki-src/landscape.md:246-247`) and the `verify` parameter text
   (`LandscapeHandler.cpp:2471`), both of which should say the counters are texel counts and name
   the magnitude fields as the ones that answer "how much".

Cheaper interim if (2) is deferred: state the unit in the `verify` description and the wiki table.
That is strictly worse — it documents the gap instead of closing it — but it is better than the
current text, which asserts the counter measures the thing it cannot.

## Distinct from

- **`B-paint-blanks-unallocated-layers-component-wide`** (OPEN, High) — **the closest neighbour,
  same counter, and deliberately filed as a second ticket rather than an encounter on it.** That
  ticket establishes one collapse in the census's alphabet: `GetWeightDataFast` returns byte 0 for
  **both** "the component has no allocation for this layer" and "allocated, weight 0", so a
  per-component allocation change — which is what flips the shader permutation and blanks the
  groundcover — is not representable. This is a **third** state collapse on the same read:
  everything from 1 to 255 is one symbol, so a magnitude change is not representable either.

  **The split test is that ticket's own sentence:** *"A perfect, orphan-aware, all-layers texel
  census — which is exactly what waves 2/3 built — is still structurally blind to it."* It says, in
  its own words, that fixing the census does not fix it. Concretely, neither fix closes the other:
  summing weight magnitude at `:2908` gives you nothing about which components gained an
  allocation, because that lives in a different data source (`GetWeightmapLayerAllocations`, walked
  at `:2343-2344`, ~550 lines away in a different function) and is not a weightmap read at all;
  and adding that ticket's `componentsGainingFirstAllocation` /
  `layersNowSampledZeroOnTouchedComponents[]` leaves `otherLayerTexelsLost` exactly as insensitive
  to 255 -> 1 as it is now. Two seats, two data sources, two independently landable fixes. They
  rate a band apart as well, which is the second reason not to bundle: that one is High because the
  damage is visible in the frame today and the response certifies clean over it; this one is
  Medium because on the default path there is currently nothing to miss.

  **Sequencing note for a fixer who takes both:** do the allocation term first. It needs a new
  walk-and-diff; this one is a second accumulator in a loop that already runs, so it lands
  cleanly on top rather than the other way round.
- **`B-paint-auto-created-layer-never-weight-blended`** (OPEN, High) — the reason this ticket is
  latent today, per § *Currently latent* above. Not a blocker (see § *Not deferred*), but whoever
  fixes it makes this one live, and the two should be read together.
- **`B-layer-paint-doc-claims-blend-group-write`** (OPEN, Medium) — its `#1` named this exact
  blindness as the limit of its own measurement (*"a merge-time renormalization that took Rock from
  255 to 127 would leave the identical 2601 and the identical `otherLayerTexelsLost: 0`"*) and set
  an escalation condition on it: *"if the census's texel-count metric is shown to hide a real
  weight reduction, the response stops being honest and this becomes High."* Recorded there in `#2`
  and repeated here for symmetry: **that condition is not met today**, because there is no
  reduction to hide, so that ticket stays Medium. It fires when
  `B-paint-auto-created-layer-never-weight-blended` lands.
- **`B-paint-erases-orphaned-layer`** (IN-REVIEW, High) — the census's registration-side blindness:
  a layer absent from `LandscapeInfo->Layers` is skipped at `:2927-2930` before any read happens.
  A fourth, separate gap on the same instrument, and already owned.
- **`B-paint-layer-destroys-other-layer-weights`** (IN-REVIEW, Critical) — **cited, not touched.**
  Its `#5` is where this blindness was first written down, as an aside on an axis that ticket
  never claimed; that entry explicitly left it for a separate ticket, which this is. Its verdict —
  the erasure was real on the build it was filed against and is fixed by waves 2/3 — is settled and
  is not reopened here.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* — *the call succeeds, every number it
reports is correct, and the output is wrong because the deciding number was never reported.*
**A member, with the qualification that makes it the mildest one on the board right now.** Every
number is correct in the strictest sense: `otherLayerTexelsLost: 0` is arithmetically exact for the
quantity it computes. The deciding number — how much weight moved — is never computed, from a
buffer that already holds it, in a loop that already visits every byte of it. What keeps this from
rating with its siblings in the class is that the wrong output it would license is not currently
produced on this project: see § *Currently latent*.

## Dedup

Board-wide grep for `SampleLayerWeights` (3 files), `otherLayerTexelsLost` (4), `GetWeightDataFast`,
`census` and `create_procedural_terrain` (20), with every hit's frontmatter read and the three
census-bearing tickets read in full. The census's known gaps are owned as follows: the
registration-side gap by `B-paint-erases-orphaned-layer`, the allocation-side gap by
`B-paint-blanks-unallocated-layers-component-wide`, the doc claims about the census by
`B-layer-paint-doc-claims-blend-group-write`. **The magnitude gap is named nowhere as a ticket** —
it appears only as an aside inside `B-paint-layer-destroys-other-layer-weights` `#5`, which
explicitly deferred it, and as the un-escalated condition in
`B-layer-paint-doc-claims-blend-group-write` `#1`.

## Not done

No source modified, no test written, no editor call made (pid 18592 untouched). **No measurement
was taken for this ticket** — the finding is a source derivation, and the fixture that would
demonstrate it (a landscape whose layers are genuinely `FinalWeightBlending`, per § *Not deferred*)
was not built. The 255 -> 1 figure is an illustration of what the predicate at `:2908` admits, not
an observed value. The proposed fields are not prototyped.

severity rationale: impact=Medium — the README's soft-blocker band, *"a readback omits a field and forces a fallback"*, which is this case almost verbatim: the response reports the sibling cost in the wrong unit, so a caller who needs the magnitude must fall back to reading weightmaps themselves, and the fallback is expensive (no PinWright verb returns raw weight values for a non-painted layer, so it means Landscape Ed Mode or a source dive) × reach=normal — **both modifiers declined**. No bump up: the census runs on every `verify: true` paint, which is the default, but layer painting is terrain authoring, not an almost-every-session activity — the same reading taken on the two neighbouring landscape tickets, kept identical so the three sort predictably. No bump down: the affected path is the default one (`verify` defaults true), not an edge. High is declined, and this is the argument that decides the band: High is *"silent false-success, or silent wrong data on a normal path (the caller trusts a result that is a lie and builds on it)"*, and today the counter is not a lie — on the default path nothing is weight-blended (`B-paint-auto-created-layer-never-weight-blended`), so no renormalisation occurs and `0` is the true answer. The defect is that the instrument *could not* report the loss if there were one, which is a precision gap, not a false statement. **The escalation is named and dated to an event rather than a guess: this becomes High the moment `B-paint-auto-created-layer-never-weight-blended` reaches DONE**, because from then on a strength-1.0 paint really does drive siblings toward zero and both counters certify clean over it — at which point the caller is trusting a number that is wrong, which is the High band verbatim. It is deliberately NOT rated High pre-emptively: rating a latent defect by its future state would order it above `B-paint-blanks-unallocated-layers-component-wide`, whose damage is visible in the frame today. Low is declined because this is not naming or docs — renaming the field would not let anyone measure the magnitude; the value is never computed, from a buffer that already contains it -> Medium

## History
- `#1-census-counts-presence-not-magnitude` `OPEN` reporter — **`SampleLayerWeights` tallies `if (Weights[X + Y * SizeX] == 0) { continue; }` (`LandscapeHandler.cpp:2908`), so a sibling layer renormalised from 255 down to 1 is still non-zero, still counted, and contributes 0 to both `otherLayerTexelsLost` (`:3074`) and `otherLayerTexelsLostInRegion` (`:3075`). 99.6% of a layer's weight can leave the painted region with both counters reading 0.** Re-derived at plugin HEAD `1a9e5778` (`LandscapeHandler.cpp` clean against HEAD); every cited line opened for this filing: `:2893` (the lambda, whose own comment names its unit — *"texels carrying any weight"*), `:2904-2905` (the single `GetWeightDataFast` it is built on), `:2908` (the entire discrimination), `:2911-2916` (the inside/outside split of that same boolean), `:2927-2930` (census set), `:3074`/`:3075` (accumulation), `:3082-3088` (the warning, raised only on the texel term), `:3139`/`:3140` (publication). The painted layer's own readback shares the shape: `texelsWithWeight` counts `Sample != 0` and only `texelsAtRequestedWeight` carries magnitude, for one layer at one value. **Why it matters here specifically:** magnitude is the exact quantity this verb's contract is about — the registered summary at `:2463` says painting at strength 1.0 *"drives every other layer in that blend group toward zero"* and sells the census as making that *"a reported number instead of a discovery in Landscape Ed Mode"*, and `Docs/wiki-src/landscape.md:246` labels the counter *"Weight the other layers lost"*. On a landscape whose layers really are `FinalWeightBlending`, that renormalisation drives siblings *toward* zero without reaching it, and both counters read 0 through the whole thing. **Currently latent, stated plainly rather than glossed:** nothing on this project's default path is weight-blended — every auto-created `ULandscapeLayerInfoObject` lands on `ELandscapeTargetLayerBlendMethod::None` (`LandscapeLayerInfoObject.cpp:27` -> `LandscapeSettings.h:167`, unset in this project's `Config/` and the engine's) and the merge's final weight-blending pass is gated on `FinalWeightBlending` (`LandscapeEditLayers.cpp:791`, gate `:3594`) — so no renormalisation occurs and `0` is presently the true answer. That is `B-paint-auto-created-layer-never-weight-blended`, filed in the same pass; this counter starts certifying clean over real loss the moment that lands. **`blockedBy` considered and REJECTED, on a measurement rather than a judgement:** a weight-blended fixture is reachable through PinWright today without any fix, because `material.authoring.add_landscape_layer` with `noWeightBlend: false` explicitly passed calls `SetBlendMethod(FinalWeightBlending)` (`MaterialAuthoringHandler.cpp:3211`) — the only blend-method write in the plugin. A failing test is writable and runnable now, so the README's requirement that a defer name a real gate is not met and parking this would be wrong. **Filed as a second ticket rather than an encounter on `B-paint-blanks-unallocated-layers-component-wide`, and the split test is that ticket's own sentence** — *"A perfect, orphan-aware, all-layers texel census ... is still structurally blind to it."* Concretely neither fix closes the other: summing weight magnitude at `:2908` reveals nothing about which components gained an allocation (that lives in `GetWeightmapLayerAllocations`, walked at `:2343-2344`, a different function and a different data source), and adding that ticket's `componentsGainingFirstAllocation` / `layersNowSampledZeroOnTouchedComponents[]` leaves this counter exactly as insensitive to 255 -> 1. Two seats, two data sources, two bands. Cross-linked both ways; sequencing advice recorded there and here — take the allocation term first, since this one is a second accumulator in a loop that already runs. **Fix:** sum the byte alongside the existing count in the same loop (zero extra reads), publish `otherLayerWeightLost` / `otherLayerWeightLostInRegion` and per-layer `weightBefore`/`weightAfter` in `layersAffected[]` **in addition to** the texel counts (which three other tickets reason about and which answer a different, still-useful question), raise the outside-region warning on the weight term too, and correct the two naming sites that invite the over-read (`Docs/wiki-src/landscape.md:246-247`, `LandscapeHandler.cpp:2471`). **Dedup:** board-wide grep for `SampleLayerWeights`, `otherLayerTexelsLost`, `GetWeightDataFast`, `census`, `create_procedural_terrain`, with the three census-bearing tickets read in full — the registration-side gap is owned by `B-paint-erases-orphaned-layer` (`:2927-2930` skips unregistered layers before any read), the allocation-side gap by `B-paint-blanks-unallocated-layers-component-wide`, the doc claims by `B-layer-paint-doc-claims-blend-group-write`; the magnitude gap is named nowhere as a ticket, appearing only as a deferred aside in `B-paint-layer-destroys-other-layer-weights` `#5` and as the un-escalated condition in `B-layer-paint-doc-claims-blend-group-write` `#1`. That Critical is cited and deliberately **not** edited; its settled verdict is not reopened. **Severity `Medium`** with High declined on the ground that decides the band — today the counter is not a lie, because there is no loss to miss, so this is a precision gap rather than silent wrong data — and the escalation tied to an event rather than a guess: **it becomes High when `B-paint-auto-created-layer-never-weight-blended` reaches DONE**. Rating it High pre-emptively would order it above damage that is visible in the frame today. Low declined: renaming the field would not let anyone measure the magnitude, which is never computed from a buffer that already holds it. Reach declined both ways. **Not done:** no source modified, no test, no editor call, and **no measurement taken** — the finding is a source derivation, the weight-blended fixture was not built, and the 255 -> 1 figure illustrates what the predicate admits rather than reporting an observed value.
