---
id: B-layer-paint-doc-claims-blend-group-write
title: "landscape.create_procedural_terrain's docs attribute Landscape Ed Mode's blend-group stroke to a verb that writes exactly one channel — the brush half of the claim is true, the verb half is false, and the same generated page states the correct mechanism two lines above it"
status: OPEN
severity: Medium
category: bug
tags: [landscape, create_procedural_terrain, docs, wiki-src, weightmap, layer-paint, blend-group, setalphadata, engine-parity, self-contradicting-page, registry-description]
encounters: 2
lastSeen: 2026-08-30T16:35:00+03:00
---

# The page says painting a layer costs its siblings; the write touches one channel

`landscape.create_procedural_terrain` paints one weight-blended target layer. Its documentation
tells the caller that doing so also writes every *other* layer in the same blend group down, and
that this is what Landscape Ed Mode's brush does. **The Ed Mode half is true. The verb half is
false**, and the verb's own page says so 2 lines earlier.

## The two sentences, on one generated page, two lines apart

`Saved/PinWright/wiki/landscape.create_procedural_terrain.md:41` (from
`Docs/wiki-src/landscape.md:237`), the parenthetical that closes the response-fields paragraph:

> (`SetAlphaData` itself does not renormalize — it writes only the named layer's channel. The
> renormalization is the edit-layer merge's final weight-blending pass.)

`Saved/PinWright/wiki/landscape.create_procedural_terrain.md:43` (from
`Docs/wiki-src/landscape.md:239`), in full because every clause of it is load-bearing:

> **Painting one layer costs the other layers in the same blend group — inside the painted
> region.** Weight-blended layers normalize to a sum of 1, so `strength: 1.0` on layer A
> necessarily drives every sibling toward 0 wherever A was written. That is the engine's
> behaviour, and it is the same thing Landscape Ed Mode's paint brush does: the engine's own
> stroke enumerates the target layer's blend group and writes *every* member of it, the painted
> one up and the siblings down, strictly inside the brush bounds. The consequence for callers:
> **build multi-layer terrain by painting each layer over the region it should own** — never by
> painting one layer over the full extent and then a second one on top, which erases the first
> everywhere they overlap.

One paragraph says the write touches one channel and the normalization happens elsewhere; the
next says the write enumerates the blend group and moves the siblings. They are not two readings
of one fact — they name different code.

## The brush half is TRUE, re-derived at UE 5.8

Worth establishing first, because it is what makes the sentence plausible and what a fixer will
otherwise re-derive from scratch.

`FLandscapeToolStrokePaint::GetAffectedTargetLayersForTarget`
(`C:/UE_5.8/Engine/Source/Editor/LandscapeEditor/Private/LandscapeEdModePaintTools.cpp:198-224`)
takes the targeted layer, asks `ULandscapeInfo::GetBlendGroupForTargetLayer` for its blend group,
and adds every member that is not the target as `FAffectedTargetLayer(LayerInfo, /*bInInverted =
*/true)` at `:217`. `Apply` then loops **all** of them at `:315`, skipping only those whose
`GetTargetLayerActionType` (`:149-166`) returns `None`, and each affected layer has its own cache
that flushes through `TAlphamapAccessor::SetData`
(`C:/UE_5.8/Engine/Source/Runtime/Landscape/Public/LandscapeEdit.h:553`). So the brush achieves
the blend-group write by looping layers, one single-layer `SetAlphaData` per layer.

The inverted siblings are written only when *exclusive painting* is on — and it is on by default:
`WeightBlendedTargetLayerPaintMode` is read with `InDefaultValue =
ELandscapeWeightBlendPaintMode::ExclusiveRequiresNoCtrl`
(`C:/UE_5.8/Engine/Source/Editor/LandscapeEditor/Private/LandscapeEditorObject.cpp:368`), and
`IsExclusivePaintingRequested` (`LandscapeEdModePaintTools.cpp:226-251`) returns
`!bAlternateModifierPressed` for that mode. A default brush stroke with no Ctrl held writes the
siblings down.

## The verb half is FALSE, re-derived at plugin HEAD `1a9e5778`

`landscape.create_procedural_terrain` does not go near any of that. Its one write is
`LandscapeEdit.SetAlphaData(LayerInfo, PaintMinX, PaintMinY, PaintMaxX, PaintMaxY,
AlphaData.GetData(), RegionSizeX)` at
`Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp:3010-3011` — one
`ULandscapeLayerInfoObject`, resolved once, no blend-group enumeration anywhere in the handler.

That overload is
`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeEditInterface.cpp:2106`, and its
per-texel write is `uint8& Weight = LayerDataPtrs[UpdateLayerIdx][TexDataIndex]; Weight =
NewWeight;` at `:2324-2325`, with `LayerEditDataAllZero[UpdateLayerIdx] = false` at `:2328`.
`UpdateLayerIdx` is the requested layer's allocation index and is the only index the loop
subscripts — no sibling channel is read or written. The 10-argument overload that once carried
`bWeightAdjust` / `bTotalWeightAdjust` is marked `// Deprecated` at `:2377` and discards both,
forwarding to the single-layer one (`:2378-2381`).

**The handler already knows.** `LandscapeHandler.cpp:3054-3058`, a comment in this same file:

> NOTE the correction: SetAlphaData itself does NOT renormalize. On 5.8 the single-layer overload
> (LandscapeEditInterface.cpp:2106) writes only LayerDataPtrs[UpdateLayerIdx] and never touches a
> sibling channel, and the 10-argument overload (:2378) discards bWeightAdjust /
> bTotalWeightAdjust and forwards to it. The renormalization is the edit-layer merge's, not the
> write's.

## The claim is on the wire as well as in the wiki, and in a third place in source

Three sites carry it, and a fixer must sweep all three or the tool description will keep
contradicting the page:

1. `Docs/wiki-src/landscape.md:239` — the overlay, and the fixable source. It reaches
   `Saved/PinWright/wiki/landscape.create_procedural_terrain.md:43`.
2. `LandscapeHandler.cpp:2463`, the `REGISTER_RPC_HANDLER` description — *"COSTS THE SIBLING
   LAYERS: weight-blended layers normalize as a group, so painting one at strength 1.0 drives
   every other layer in that blend group toward zero WITHIN the painted region — that is the
   engine's behaviour, not a defect"*. This is the verb's tool description: every MCP client that
   lists the `landscape` namespace reads it, and it is echoed into
   `Saved/PinWright/wiki/landscape.md:197`.
3. `LandscapeHandler.cpp:2865-2872`, the comment justifying the census, which states the Ed Mode
   mechanism correctly and then draws the wrong conclusion for this verb: *"So siblings losing
   weight INSIDE the region is the engine's normalization working as designed and is reported as
   a number, not a warning."* This one sits ~190 lines above the correction quoted earlier, in the
   same function.

## Measured, and the exact limit of the measurement

**13:32 build, plugin commit `d8f1bc32`** (an ancestor of HEAD; `LandscapeHandler.cpp` and
`Docs/wiki-src/landscape.md` are byte-identical at `d8f1bc32` and `1a9e5778`, so the measurement
applies to HEAD's text). Source of record, read and not relayed:
`X:/src/unreal/EAContentExamples58/Docs/map/vegetation-test-level.md:304-317`, project commit
`7d629ad9` *"Re-measure the vegetation map against the 13:32 plugin build"*.

Painting `Grass` at strength 1.0 over the **full extent** left `Rock` at
`texelsWithWeightBefore 2601 -> After 2601`. Across six paints in that session
(`Grass` full extent, `Grass` 51x51, `Rock` 51x51, two strength-0 erases)
`otherLayerTexelsLost` was `0` every time.

**What that refutes:** the doc's *"erases the first everywhere they overlap"*. Rock's 2601 texels
lie inside the extent Grass was painted over; erasure would have taken the count to 0. It did not
move.

**What it does NOT refute, stated so nobody over-reads it:** *"drives every sibling toward 0"*.
The census counts **texels carrying any weight**, not weight values — `SampleLayerWeights` is
introduced by the comment *"Tallies one layer's texels carrying any weight over the FULL extent"*
(`LandscapeHandler.cpp:2891-2892`), and the painted-layer readback tallies `if (Sample != 0)
{ ++TexelsWithWeight; }` at `:3044-3047`, with `OtherLayerTexelsLost` accumulating
`OutsideBefore - OutsideAfter` at `:3074`. A merge-time renormalization that took Rock from 255
to 127 would leave the identical 2601 and the identical `otherLayerTexelsLost: 0`. The
edit-layer merge's final weight-blending pass
(`FLandscapeEditLayersWeightmapsPerformFinalWeightBlendingPS`, named at
`LandscapeHandler.cpp:3050-3051`) plausibly does exactly that. **Untested here, and this ticket
does not claim it either way.**

So the defensible statement is the narrow one: **the write is single-channel, so the caller must
paint the siblings to strength 0 themselves to make a layer own a region** — which is what the
same session found by doing it (`vegetation-test-level.md:328-329`: the step that worked was
`Grass 0` **then** `Rock 1.0` over that rectangle, not `Rock 1.0` alone).

## Why the doc being wrong costs something

The doc's recipe — *"build multi-layer terrain by painting each layer over the region it should
own"* — is sufficient only on virgin weightmaps. The moment a region already carries another
layer's weight, following it leaves both layers carrying weight there, and the caller is
explicitly told the second paint erased the first. The step the doc omits (paint the siblings to
0 over the region first) is reachable only by measuring the census across a sibling or by reading
`LandscapeEditInterface.cpp:2324`. That is the rubric's Medium band verbatim: doable, but only
via a source dive.

## Distinct from `B-paint-layer-destroys-other-layer-weights`, and what this bears on it

`B-paint-layer-destroys-other-layer-weights` (IN-REVIEW, Critical) is about the **write reaching
outside the region** — siblings zeroed landscape-wide. This ticket is about the **documentation
of the write inside the region**, and asks for no code change to the paint path. They are
separable: correcting these three doc sites lands without touching that ticket's mechanism
question, and fixing that ticket does not correct a sentence about Landscape Ed Mode.

Two findings above bear on it and are recorded here rather than edited into it, because its
premise is under separate re-derivation right now:

- Its premise did not reproduce on the 13:32 build — `otherLayerTexelsLost` 0 across six paints,
  every loss confined to the requested rectangle (`vegetation-test-level.md:304-310`).
- **Both its premise and that non-reproduction ride on the same metric**, and the metric counts
  texels rather than weight (`LandscapeHandler.cpp:2891-2892`, `:3044-3047`, `:3074`). A
  renormalization that reduces every sibling without zeroing any is invisible to it in both
  directions. Whoever re-derives that ticket should read the weight *values* back, not the
  counts.

## Fix

Correct the claim at all three sites, saying what the verb does rather than what the brush does:

- the write is one channel — `FLandscapeEditDataInterface::SetAlphaData`, single-layer overload;
- Landscape Ed Mode's brush is **not** the same operation: it enumerates the blend group and
  writes the siblings down, per layer, and does so by default
  (`LandscapeEdModePaintTools.cpp:198-224`, `:315`, `LandscapeEditorObject.cpp:368`);
- to make a layer own a region through this verb, paint the siblings to `strength: 0` over that
  region first;
- keep `landscape.md:237`'s parenthetical — it is the sentence that is already right — and make
  `:239` agree with it instead of contradicting it.

The alternative fix is the parity one: give the verb the brush's blend-group semantics behind an
explicit parameter (`exclusive: true`, defaulting off), which would make the current sentence
true instead of deleting it. That is a real feature and a much larger change; it is named here so
the doc fix is a deliberate choice rather than the only option anyone considered. If it is taken,
it wants its own `F-` ticket and this one closes as its doc half.

## Structure: why this is its own ticket

Filed alongside `B-capture-docs-prescribe-retired-grass-wait` (docs prescribing a grass wait the
settle path bypasses) rather than merged with it. Both are the same *shape* — a shipped change
rewrote one part of a wiki page and left the older, now-false part standing on the same page —
but they share no verb, no namespace, no overlay file and no fixer, and they rate differently.
The board's one multi-defect precedent, `B-bulk-rename-docs-behaviour-mismatch`, bundles three
contradictions that live in **one verb, one function, one contiguous block**; this set is the
opposite. Bundling would also force one severity onto items the picker should order separately.

## Same shape as

Not a member of the session's recurring class stated on `B-foliage-paint-does-no-ground-projection`
§ *Same shape as* (*the call succeeds, every number it reports is correct, and the output is wrong
because the deciding number was never reported*) — here the response is honest and the census
reports the sibling cost as a number; only the prose is wrong. Said explicitly so a reader does
not file it under that class by habit.

The nearer neighbour is `B-bulk-rename-docs-behaviour-mismatch` (IN-REVIEW, High) — a verb whose
documentation describes behaviour it does not have. The difference in rating is the same
difference as here: there the docs mislead about *what the call returns*, so a caller acts on a
wrong result; here the docs mislead about *what a second call is for*, so a caller omits a step.

## Dedup

Board-wide search for `blend group`, `SetAlphaData`, `create_procedural_terrain`, sibling-layer
and weightmap-normalization tickets. The `landscape.*` files touching layer weight are
`B-paint-layer-destroys-other-layer-weights` (the write's reach — carved out above),
`B-create-procedural-terrain-paints-nothing` (DONE — false success when the layer is not on the
material, a different claim), `B-create-procedural-density-writes-paint-density`,
`E-create-procedural-terrain-no-material-echo`, `E-create-procedural-terrain-no-label-set` and
`F-landscape-paint-region-shapes` (region geometry). `B-blockout-review-ortho-snap-contradiction`
(IN-REVIEW, Medium) is the closest structural precedent — a docs-only ticket for a page that
misstates a verb's behaviour. **Nothing on the board mentions the blend-group claim, the Ed Mode
brush comparison, or the single-channel write.**

## Severity

**Medium.** Impact class: the rubric's *"soft blocker — doable, but only via a documented
workaround, a source dive, or many extra calls"*. Multi-layer terrain through this verb is
reachable, and the step that makes it work (paint the siblings to 0 first) is stated nowhere in
the docs; the docs state the opposite and attribute it to the engine.

**Low argued and declined.** The Low band covers *"docs, discoverability, naming"*, and this is
literally a doc defect. It is declined because the doc does not merely fail to help — it
prescribes a recipe that omits a required step and asserts the step is unnecessary, so a caller
who follows it produces terrain with two layers contending in the overlap and no reason to
suspect it. A Low-band doc gap leaves the caller uninformed; this one leaves them confidently
wrong.

**High argued and declined.** The High band is silent wrong data *on the wire*. The verb's
response is honest here: `layersAffected[]`, `otherLayerTexelsLost` and
`otherLayerTexelsLostInRegion` are measured and reported, and they reported the truth in the
session that found this (0 losses, six paints). Only the prose lies, and the caller has a
truthful number in front of them if they read it. Note the one thing that would flip this: if the
census's texel-count metric is shown to hide a real weight reduction (see the second bullet under
*Distinct from* above), the response stops being honest and this becomes High.

**Reach modifier declined in both directions, and named.** No bump up: `landscape.create_procedural_terrain`
is the only layer-paint verb the plugin has, but layer painting is a terrain-authoring activity,
not an almost-every-session one. No bump down: it is not a rare edge path — the sentence being
corrected is the one that tells callers how to build multi-layer terrain at all, and it also sits
in the tool description every client sees when it lists the namespace.

## History
- `#1-page-attributes-the-brush-stroke-to-the-verb` `OPEN` reporter — `Docs/wiki-src/landscape.md:239` (generated `landscape.create_procedural_terrain.md:43`) tells the caller that painting one layer at `strength: 1.0` "necessarily drives every sibling toward 0 wherever A was written", that this "is the same thing Landscape Ed Mode's paint brush does", and that painting a second layer over a first "erases the first everywhere they overlap". Re-derived at plugin HEAD `1a9e5778` and UE 5.8, both halves separately. **The Ed Mode half is TRUE and is recorded so nobody re-derives it:** `FLandscapeToolStrokePaint::GetAffectedTargetLayersForTarget` (`LandscapeEdModePaintTools.cpp:198-224`) enumerates the target's blend group and adds every non-target member as `FAffectedTargetLayer(LayerInfo, bInInverted = true)` at `:217`; `Apply` loops all of them at `:315` skipping only `ELandscapeTargetLayerActionType::None` (`:318`, decided at `:149-166`); each affected layer flushes through its own `TAlphamapAccessor::SetData` (`LandscapeEdit.h:553`), so the brush does the blend-group write by looping layers; and exclusive painting — the mode that lets the inverted siblings through — is the default, `ExclusiveRequiresNoCtrl` at `LandscapeEditorObject.cpp:368` with `IsExclusivePaintingRequested` returning `!bAlternateModifierPressed` at `:226-251`. **The verb half is FALSE:** the handler's only write is the single-layer `SetAlphaData` at `LandscapeHandler.cpp:3010-3011`, whose per-texel loop subscripts only `LayerDataPtrs[UpdateLayerIdx]` (`LandscapeEditInterface.cpp:2324-2325`, `:2328`), and the 10-argument overload that once took `bWeightAdjust`/`bTotalWeightAdjust` is `// Deprecated` at `:2377` and discards both (`:2378-2381`). No blend-group enumeration exists anywhere in the handler. **The same generated page already says so two lines earlier**, at `:41` (from `landscape.md:237`): "(`SetAlphaData` itself does not renormalize — it writes only the named layer's channel...)", and `LandscapeHandler.cpp:3054-3058` carries the same correction as a code comment. Three sites carry the false claim and all three need sweeping: the overlay `landscape.md:239`, the `REGISTER_RPC_HANDLER` description at `LandscapeHandler.cpp:2463` (which is the tool description every MCP client reads, echoed to `Saved/PinWright/wiki/landscape.md:197`), and the census-justifying comment at `LandscapeHandler.cpp:2865-2872`, 190 lines above the correction in the same function. **Measured on the 13:32 build (`d8f1bc32`, an ancestor of HEAD; `LandscapeHandler.cpp` and `Docs/wiki-src/landscape.md` are byte-identical at both commits, so the measurement applies to HEAD's text), source of record `X:/src/unreal/EAContentExamples58/Docs/map/vegetation-test-level.md:304-317` at project commit `7d629ad9`, read directly:** painting `Grass` 1.0 over the full extent left `Rock` at `texelsWithWeightBefore 2601 -> After 2601`, and `otherLayerTexelsLost` was 0 across six paints. **The limit of that measurement is stated rather than glossed:** it refutes "erases" (Rock's texels are inside the painted extent; erasure would have taken the count to 0) but NOT "drives toward 0", because the census counts texels carrying any weight and not weight values — `SampleLayerWeights`' own comment at `LandscapeHandler.cpp:2891-2892`, the painted-layer tally `if (Sample != 0)` at `:3044-3047`, and `OtherLayerTexelsLost += OutsideBefore - OutsideAfter` at `:3074` — so a merge-time renormalization from 255 to 127 would produce the identical numbers. That is left open and is not claimed either way. The defensible statement is that the write is single-channel and the caller must paint the siblings to strength 0 to make a layer own a region, which is what the same session found empirically (`vegetation-test-level.md:328-329`). Carved out of `B-paint-layer-destroys-other-layer-weights` (IN-REVIEW, Critical), which owns the write's reach OUTSIDE the region and is under separate re-derivation; that file was deliberately not edited. Two findings are recorded here for whoever re-derives it: its premise did not reproduce on this build, and both its premise and that non-reproduction ride on the same texel-count metric, which cannot see a reduction that does not reach zero — read weight values, not counts. Rated **Medium** on the source-dive band, with Low declined (the doc does not merely omit — it asserts the omitted step is unnecessary) and High declined (the response is honest; only the prose is not), and the escalation condition named: if the census metric is shown to hide a real weight reduction, the response stops being honest and this becomes High. Reach declined both ways. Filed as its own ticket rather than merged with `B-capture-docs-prescribe-retired-grass-wait`, whose shape it shares but whose verb, namespace, overlay file, fixer and rating it does not. Dedup: board-wide search for `blend group`, `SetAlphaData`, `create_procedural_terrain`, sibling-layer and weightmap-normalization tickets found nothing mentioning the blend-group claim, the Ed Mode comparison or the single-channel write; `B-blockout-review-ortho-snap-contradiction` is the closest structural precedent as a docs-only ticket about a page misstating a verb.
- `#2-root-cause-the-blend-method-is-none` `OPEN` reporter — **Additional evidence: the root cause, and it closes the one question `#1` deliberately left open.** `#1` established *that* the docs are wrong and gave one reason — the write is single-channel (`LandscapeHandler.cpp:3010-3011` -> `LandscapeEditInterface.cpp:2324-2325`) — while explicitly declining to claim anything about the merge: *"a merge-time renormalization that took Rock from 255 to 127 would leave the identical 2601 and the identical `otherLayerTexelsLost: 0` ... Untested here, and this ticket does not claim it either way."* **There is a second, independent reason, and it is the merge's: the blend-group normalisation pass is skipped outright, because every `ULandscapeLayerInfoObject` this verb auto-creates has `BlendMethod == ELandscapeTargetLayerBlendMethod::None`.** Re-derived at plugin HEAD `1a9e5778` and UE 5.8, every citation opened and read here rather than relayed. The final weight-blending shader `FLandscapeEditLayersWeightmapsPerformFinalWeightBlendingPS` is declared at `LandscapeEditLayers.cpp:791`, and the per-layer flag that admits a layer to it is `if (InLayerInfo->GetBlendMethod() == ELandscapeTargetLayerBlendMethod::FinalWeightBlending) { WeightmapTargetLayerInfo.Flags |= EWeightmapTargetLayerFlags::IsWeightBlended; }` at `:3594`. A freshly constructed `ULandscapeLayerInfoObject` takes its method from project settings in its own constructor — `BlendMethod = GetDefault<ULandscapeSettings>()->GetTargetLayerDefaultBlendMethod();` (`LandscapeLayerInfoObject.cpp:27`, accessor at `LandscapeSettings.h:87`) — and that setting's compiled default is `ELandscapeTargetLayerBlendMethod TargetLayerDefaultBlendMethod = ELandscapeTargetLayerBlendMethod::None;` (`LandscapeSettings.h:167`). **Confirmed unset rather than assumed:** the class is `UCLASS(config = Engine, defaultconfig, ...)` (`LandscapeSettings.h:45`), and grepping `X:/src/unreal/EAContentExamples58/Config/`, `X:/src/unreal/EAContentExamples58/Saved/Config/` and `C:/UE_5.8/Engine/Config/` for both `TargetLayerDefaultBlendMethod` and the `[/Script/Landscape.LandscapeSettings]` section returns **zero hits in all three**, so the compiled `None` is what every new LayerInfo gets on this host. `None` is not a neutral value: it is `None = 0 UMETA(DisplayName = "No Weight Blending", Tooltip = "The target layer's weight is unaffected by other target layers.")` (`LandscapeEditTypes.h:34`). And the **effective default flipped in 5.7**: `PostLoad` maps a pre-`LandscapeAdvancedWeightBlending` asset as `BlendMethod = bNoWeightBlend_DEPRECATED ? None : FinalWeightBlending` (`LandscapeLayerInfoObject.cpp:276`, deprecation contract at `LandscapeLayerInfoObject.h:80-82`), so a layer info authored before 5.7 with weight blending on stays weight-blended, while one constructed today does not — which is exactly why a sentence that was true when it was written is false now. **Nothing in PinWright sets it back:** `LandscapeHandler.cpp` contains **zero** `BlendMethod` references, and the only occurrence anywhere in `Source/` is `MaterialAuthoringHandler.cpp:3211`, in a different verb and behind an explicit parameter. **Measured, and attributed rather than asserted:** the discriminating measurement is not mine — it is the four-paint live run recorded as `#5-independently-reproduced-green-and-claim-a-refuted` on `B-paint-layer-destroys-other-layer-weights` (board commit `9eed329`), read there in full. Its paint (3) read back `texelsAtRequestedWeight: 256` for `Sand` over a 16x16 region — all 256 texels at exactly 255 — while the same call's `layersAffected[]` showed `Grass` and `Rock` still carrying weight in those same texels. A per-texel sum that far above 255 is a state a normalised weightmap cannot hold, so the group normalisation demonstrably did not run. The running editor's DLL is the 13:32 build, plugin `d8f1bc32`, and `git diff --name-only d8f1bc32..HEAD -- '*.cpp' '*.h'` is empty, so that measurement was taken against the same code as every line cited above. **What this changes in this ticket, and what it does not.** It does **not** change the ask: the three sites `#1` named still need correcting, and the correction is now stronger, because the false sentence has two independent refutations instead of one. It **does** answer `#1`'s open clause: *"drives every sibling toward 0"* does not merely fall below the census's resolution on this verb's default path — **it does not happen at all**. **Severity stays `Medium`, and the escalation condition `#1` named is NOT met.** That condition was *"if the census's texel-count metric is shown to hide a real weight reduction, the response stops being honest and this becomes High."* The metric's magnitude blindness is real and is now filed in its own right as `B-paint-census-counts-texels-not-weight` — `SampleLayerWeights` tallies `if (Weights[X + Y * SizeX] == 0) { continue; }` (`LandscapeHandler.cpp:2908`), so a sibling renormalised 255 -> 1 contributes 0 to both `otherLayerTexelsLost` (`:3074`) and `otherLayerTexelsLostInRegion` (`:3075`). But on this verb's default path **there is no reduction for it to hide**, so nothing on the wire is currently a lie and the High trigger has not fired. The trigger is not dead, it has moved: it fires the moment `B-paint-auto-created-layer-never-weight-blended` lands, because from then on real renormalisation occurs and the census reports 0 over it. Recorded here so a later reader does not re-argue the band from the same clause. **The behavioural half is deliberately NOT folded in.** Nothing anywhere sets a blend method, so the auto-create silently produces a layer that can never normalise; that is filed as `B-paint-auto-created-layer-never-weight-blended` (High) for `landscape.create_procedural_terrain` and `B-add-landscape-layer-noweightblend-inverted-default` (Medium) for `material.authoring.add_landscape_layer`. The split test is this ticket's own sentence — *"this ticket ... asks for no code change to the paint path"* — and it holds in both directions: correcting these three doc sites leaves both auto-creates producing `None`, and setting a blend method leaves all three sentences still describing the Ed Mode brush. Three separately landable fixes, three seats, and the `#1` precedent against bundling across verbs and namespaces applies unchanged. **One rider for whoever sweeps this file's comments:** a *fourth* stale comment lives in `LandscapeHandler.cpp` — `:2818-2819` calls `CreateLayerEditorSettingsFor` *"a deprecated empty stub"* on 5.5+, but a recursive grep for `CreateLayerEditorSettingsFor` over `C:/UE_5.8/Engine/Source/` returns nothing at all: the symbol was removed outright, not stubbed, so the comment would mislead a backporter. Harmless (the call sits behind the pre-5.5 `#if`). It is recorded on `B-paint-auto-created-layer-never-weight-blended` rather than here because its fix seat is inside the auto-create branch that ticket rewrites, not among the three blend-group sites this one corrects; named here only so a fixer already editing comments in this file does not have to find it twice. **`lastSeen` note:** refreshed to the true local instant, offset `+03:00` — this host runs UTC+3 (`git log` on `#1`'s own commit `ed8f00c` reads `2026-08-30 16:18:08 +0300`), whereas `#1` recorded `+05:00`, which placed its stamp ~42 minutes ahead of the commit that wrote it. `encounters` bumped to 2: this entry is a fresh observation of *this* ticket's defect on a second axis, not a re-report of a sibling's.
