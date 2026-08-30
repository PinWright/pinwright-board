---
id: B-paint-auto-created-layer-never-weight-blended
title: "landscape.create_procedural_terrain auto-creates every ULandscapeLayerInfoObject with BlendMethod None, so the layer it just built can never take part in the group normalisation the verb's own registered summary sells — and because it bypasses UE::Landscape::CreateTargetLayerInfo it also bypasses the one project setting that exists to fix that"
status: OPEN
severity: High
category: bug
tags: [landscape, create_procedural_terrain, weightmap, layer-paint, blend-method, weight-blending, layerinfo-auto-create, engine-parity, silent-wrong-output, landscape-settings, ue57-regression, stale-comment]
encounters: 1
lastSeen: 2026-08-30T16:45:00+03:00
---

# The verb manufactures the layer, then never tells it it is weight-blended

`landscape.create_procedural_terrain` auto-creates a `ULandscapeLayerInfoObject` for any target
layer the material declares but no asset backs — that branch is the default path for the first
paint of every layer, and `layerInfoAutoCreated: true` is how the response says so. The object it
constructs comes out with `BlendMethod == ELandscapeTargetLayerBlendMethod::None`, whose engine
display name is literally **"No Weight Blending"**, and nothing in PinWright ever changes it.

The verb's registered summary, which is the tool description every MCP client reads, promises the
opposite in capitals:

> **COSTS THE SIBLING LAYERS:** weight-blended layers normalize as a group, so painting one at
> strength 1.0 drives every other layer in that blend group toward zero WITHIN the painted region
> — that is the engine's behaviour, not a defect
> (`Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp:2463`)

The layer this verb creates is not in a blend group in any sense the merge recognises. The
promised normalisation is not merely unreported — it **cannot occur**.

## The chain, re-derived at plugin HEAD `1a9e5778` and UE 5.8

Every line below was opened and read for this ticket, not carried over from another entry.

**1. The normalisation is a merge-time shader pass, and it is gated per layer.**
`FLandscapeEditLayersWeightmapsPerformFinalWeightBlendingPS` is declared at
`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeEditLayers.cpp:791`. A layer is
admitted to it only here:

```cpp
if (InLayerInfo->GetBlendMethod() == ELandscapeTargetLayerBlendMethod::FinalWeightBlending)
{
    WeightmapTargetLayerInfo.Flags |= EWeightmapTargetLayerFlags::IsWeightBlended;   // :3594
}
```

**2. A newly constructed LayerInfo takes its method from project settings.**

```cpp
// The default blend method comes from the settings :
BlendMethod = GetDefault<ULandscapeSettings>()->GetTargetLayerDefaultBlendMethod();
// LandscapeLayerInfoObject.cpp:27, in ULandscapeLayerInfoObject's own constructor
```

accessor at `Runtime/Landscape/Public/LandscapeSettings.h:87`.

**3. That setting's compiled default is `None`, and nothing overrides it here.**

```cpp
/** Target layer blend method to use for newly created Landscape Layer Info assets. ...
    This is only used when DefaultLayerInfoObject isn't set. */
UPROPERTY(Config, Category = "Target Layers", EditAnywhere)
ELandscapeTargetLayerBlendMethod TargetLayerDefaultBlendMethod = ELandscapeTargetLayerBlendMethod::None;
// LandscapeSettings.h:167
```

The class is `UCLASS(config = Engine, defaultconfig, ...)` (`LandscapeSettings.h:45`). **Checked
rather than assumed:** grepping `X:/src/unreal/EAContentExamples58/Config/`,
`X:/src/unreal/EAContentExamples58/Saved/Config/` and `C:/UE_5.8/Engine/Config/` for both
`TargetLayerDefaultBlendMethod` and the `[/Script/Landscape.LandscapeSettings]` section returns
**zero hits in all three**. Nothing anywhere sets it, so the compiled `None` is what every new
LayerInfo gets on this host.

**4. `None` means exactly what it sounds like.**

```cpp
None = 0 UMETA(DisplayName = "No Weight Blending",
               Tooltip = "The target layer's weight is unaffected by other target layers."),
// LandscapeEditTypes.h:34
```

**5. PinWright never sets it.** `LandscapeHandler.cpp` contains **zero** `BlendMethod`
references. Across all of `Source/`, the single occurrence is
`Handlers/Material/MaterialAuthoringHandler.cpp:3211` — a different verb, behind an explicit
parameter, and covered by its own ticket. The auto-create branch
(`LandscapeHandler.cpp:2777-2825`) does `NewObject<ULandscapeLayerInfoObject>`, `SetLayerName`,
and `LandscapeInfo->CreateTargetLayerSettingsFor(NewLayerInfo)` — and that last call only
registers, looping proxies to `AddTargetLayer` / `UpdateTargetLayer`
(`Runtime/Landscape/Private/Landscape.cpp:4478-4493`). It touches no blend method.

## The PinWright-specific half: the engine factory is bypassed, and with it the escape hatch

This is the part that is not simply "the engine's default is unhelpful", and it is why this is a
plugin defect rather than an engine one.

The engine has exactly **one** blessed factory for target-layer infos —
`UE::Landscape::CreateTargetLayerInfo` (`Runtime/Landscape/Private/LandscapeUtils.cpp:286`,
`LANDSCAPE_API` at `LandscapeUtils.h:321/330`). It is what the Target Layers panel
(`LandscapeEditorDetailCustomization_TargetLayers.cpp:2354`), the import-layers path (`:414` of
`..._ImportLayers.cpp`), `LandscapeEditorObject.cpp:962` and the world browser
(`WorldTileCollectionModel.cpp:1564`) all call. Its **first** action is:

```cpp
// Get the default asset from the project settings
const ULandscapeSettings* Settings = GetDefault<ULandscapeSettings>();
TSoftObjectPtr<ULandscapeLayerInfoObject> DefaultLayerInfoObject = Settings->GetDefaultLayerInfoObject().LoadSynchronous();
...
if (DefaultLayerInfoObject.Get() != nullptr)
{
    LayerInfo = DuplicateObject<ULandscapeLayerInfoObject>(DefaultLayerInfoObject.Get(), Package, *InFileName);
```

`DefaultLayerInfoObject` (`LandscapeSettings.h:128`, getter `:77`) is **the** project-level knob
for "make my new layer infos look like this one", and it is what the `TargetLayerDefaultBlendMethod`
doc comment defers to (*"only used when DefaultLayerInfoObject isn't set"*).

PinWright reaches none of it. `LandscapeHandler.cpp:2778-2780` calls `NewObject` directly, and
`CreateTargetLayerInfo` / `LandscapeUtils` appear nowhere in the landscape handler
(`grep -rn "CreateTargetLayerInfo" Source/` matches nothing). **So a project that configures
`DefaultLayerInfoObject` to a weight-blended template — the engine's supported, documented cure
for exactly this — still gets a raw `None` layer out of this verb.** The one control the caller
has is disconnected from the code path that needs it.

## Why the caller cannot find out

- The response publishes `layerInfoAutoCreated`, `texelsWithWeight`, `texelsAtRequestedWeight`,
  `layersAffected[]`, `otherLayerTexelsLost`, `otherLayerTexelsLostInRegion`, the orphan-scan
  fields — and **no blend method**, for the layer it just manufactured or for any sibling.
- The census is consistent with a normalised landscape *and* with this one, because it counts
  texels carrying any weight, not weight (`LandscapeHandler.cpp:2908`) — see
  `B-paint-census-counts-texels-not-weight`.
- The registered summary (`:2463`), the `verify` parameter text (`:2471`), the census-justifying
  comment (`:2865-2872`) and `Docs/wiki-src/landscape.md:239` all describe the group behaviour as
  a thing that happens. None of them mentions a blend method at all.

The only way to the fact is `LandscapeLayerInfoObject.cpp:27` plus `LandscapeSettings.h:167`, or
opening the created `LayerInfo_<Name>` sub-object inside the landscape actor's package and reading
its Blend Method — and the object is not a Content Browser asset, it is created with outer
`Landscape` (`LandscapeHandler.cpp:2778-2780`), so even that is not a two-click check.

## What it actually costs

Weights stop summing to 1. Two layers painted at `strength: 1.0` over the same texels both hold
255, which is a state a normalised weightmap cannot reach. What a `LandscapeLayerBlend` node in
`LB_WeightBlend` mode then renders is a blend fed by unnormalised inputs — not a crash, not a
loss, but not what the caller asked for or was told to expect, and the recipe the docs give for
multi-layer terrain (*"paint each layer over the region it should own"*) silently stops being
sufficient because the second paint no longer displaces the first.

**This is a 5.7-era flip, not a long-standing state.** `PostLoad` migrates pre-`LandscapeAdvancedWeightBlending`
assets as `BlendMethod = bNoWeightBlend_DEPRECATED ? None : FinalWeightBlending`
(`LandscapeLayerInfoObject.cpp:276`; deprecation contract at `LandscapeLayerInfoObject.h:80-82`),
so a layer info authored before 5.7 with weight blending on stays weight-blended. Only *newly
constructed* ones land on the settings default. Whether the pre-5.7 constructor default was
weight-blended is **not established here** — that source is not in this tree, and it is not
asserted.

## Measured — attributed, not re-run

The discriminating measurement is not this ticket's. It is the four-paint live run recorded as
`#5-independently-reproduced-green-and-claim-a-refuted` on
`B-paint-layer-destroys-other-layer-weights` (board commit `9eed329`), read there in full. Paint
(3) painted `Sand` at 1.0 over a 16x16 region and read back `texelsAtRequestedWeight: 256` — all
256 texels at exactly 255 — while the same call's `layersAffected[]` showed `Grass` and `Rock`
still carrying weight in those same texels. A per-texel sum that far above 255 cannot exist in a
normalised weightmap, so the final weight-blending pass demonstrably did not run on those layers.
That entry also establishes the readback is a genuine composited read rather than an edit-layer
one (`LandscapeEditInterface.cpp:81`, `:2641`, `Landscape.cpp:2624`).

The running editor is the **13:32 build**, plugin `d8f1bc32`; `git diff --name-only d8f1bc32..HEAD
-- '*.cpp' '*.h'` is empty, so that measurement was taken against the same code as every citation
above. **No editor call was made for this filing** and nothing was re-run.

## Fix

Two seats, either of which closes it; the first is preferable.

1. **Route the auto-create through `UE::Landscape::CreateTargetLayerInfo`** (or replicate its two
   steps: honour `ULandscapeSettings::GetDefaultLayerInfoObject()` by duplication, else construct
   and then set the method). This reconnects the project-level knob and puts PinWright on the same
   factory as the Target Layers panel, which is the parity argument for every other choice in this
   branch already.
2. **At minimum, call `SetBlendMethod(ELandscapeTargetLayerBlendMethod::FinalWeightBlending, ...)`**
   on the auto-created object — the same call `MaterialAuthoringHandler.cpp:3211` already makes —
   and expose an opt-out parameter so a caller who wants a non-blended layer can say so.

Either way, **publish the blend method in the response** (per layer in `layersAffected[]`, and for
the auto-created one), and stop the summary at `:2463` asserting group normalisation without it.
A `warnings[]` line when a paint's layer is not `FinalWeightBlending` — "this layer does not
participate in group normalisation; sibling weights will not be reduced" — is the cheapest thing
that would have made this visible at the call site.

Note for whoever takes seat 1: `LandscapeHandler.cpp` creates the object with outer `Landscape`
and no `/Game` package on purpose (a private per-landscape LayerInfo), while `CreateTargetLayerInfo`
takes a file path and makes a real asset. The blend-method half is separable from that choice —
do not let the packaging difference block the fix.

**Rider, folded in deliberately (Low on its own, and stated so):** `LandscapeHandler.cpp:2818-2819`
calls `CreateLayerEditorSettingsFor` *"a deprecated empty stub"* on 5.5+. On UE 5.8 that symbol
**does not exist anywhere** — a recursive grep for it over `C:/UE_5.8/Engine/Source/` returns
nothing; it was removed outright, not stubbed. Harmless today (the call sits behind the pre-5.5
`#if`), but it would mislead a backporter into thinking the 5.5+ path had a no-op fallback when it
has no symbol at all. Folded here rather than given a ticket because it sits **inside the very
branch this ticket rewrites** (`:2777-2825`), so a fixer is already looking at it; a standalone
Low ticket for a two-word comment edit would occupy a picker slot for something the paint-verb fix
lands for free. Named on `B-layer-paint-doc-claims-blend-group-write` `#2` as well, so a fixer
sweeping *that* ticket's three comment sites in this same file does not have to rediscover it.

## Distinct from

- **`B-layer-paint-doc-claims-blend-group-write`** (OPEN, Medium) — its `#2` carries this root
  cause and points here. That ticket is the **doc** defect: three prose sites
  (`Docs/wiki-src/landscape.md:239`, `LandscapeHandler.cpp:2463`, `:2865-2872`) describe Landscape
  Ed Mode's brush as if it were this verb, and its own body says it *"asks for no code change to
  the paint path"*. **Neither fix closes the other**, which is the split test: correcting all three
  sentences leaves the auto-create still producing `None`, and setting a blend method leaves all
  three sentences still misattributing the Ed Mode stroke. Filed separately for that reason and
  because they rate differently.
- **`B-add-landscape-layer-noweightblend-inverted-default`** (OPEN, Medium) — the sibling verb,
  same root value, **different seat and not closed by this fix**. `material.authoring.add_landscape_layer`
  calls `SetBlendMethod` only when `noWeightBlend` is explicitly present in the payload
  (`MaterialAuthoringHandler.cpp:3208-3214`), so omitting it leaves the constructor default.
  Routing *this* verb through the engine factory does not touch that handler, and fixing that
  handler's omission default does not touch this one. Two verbs, two namespaces, two files —
  the same non-bundling argument `B-layer-paint-doc-claims-blend-group-write` § *Structure* makes
  against merging across verbs.
- **`B-paint-census-counts-texels-not-weight`** (OPEN, Medium) — filed in the same pass and
  **dependent on this one in an interesting direction**: because nothing on this verb's default
  path is weight-blended, no renormalisation happens, so that ticket's blindness is currently
  **latent on this path**. It goes live the moment this ticket lands. It is deliberately *not*
  `blockedBy` this ticket — see its own § *Not deferred*.
- **`B-paint-layer-destroys-other-layer-weights`** (IN-REVIEW, Critical) — **cited, not touched.**
  Its verdict is settled: the erasure was real on the 10:44 build it was filed against and is
  fixed by waves 2/3. This ticket takes only the measurement recorded in its `#5` and the
  citations it re-derived; it makes no claim about the erasure and is not a revival of it. The
  registration fix that closed it (`CreateTargetLayerSettingsFor` at `LandscapeHandler.cpp:2815`)
  is one line above the code this ticket asks to change, and is not in question.
- **`B-paint-blanks-unallocated-layers-component-wide`** (OPEN, High) — the same verb, a
  different silent-output axis: there the damage is a per-component *allocation* change flipping
  the shader permutation. Independent of blend method entirely; a component gains or loses an
  allocation regardless of how the layer blends. Both can be true of one paint.
- **`B-configure-layer-blend-wrong-nodes`** (OPEN) and
  **`B-material-authoring-save-no-disk-write`** (IN-REVIEW) — the two other tickets that mention
  `add_landscape_layer`. Neither touches blend method: one is about the *material graph* nodes
  that declare a target layer, the other about the created asset never reaching disk. Checked
  because both would plausibly own this and neither does.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* — *the call succeeds, every number
it reports is correct, and the output is wrong because the deciding number was never reported.*
**A member, with one qualification worth stating.** The deciding value here is not merely
unreported, it is never *set*: `BlendMethod` is decided by a constructor reading an unset project
setting, and no PinWright code path ever names it. So this sits one step further out than the
class's usual form — the number the response omits is a fact about state the verb itself chose by
default and does not know it chose.

## Dedup

Board-wide grep for `BlendMethod`, `FinalWeightBlending`, `noWeightBlend`, `blend method`,
`weight-blend` and `add_landscape_layer` across all 1554 root tickets, with every hit's
frontmatter read. Hits: `B-layer-paint-doc-claims-blend-group-write` and
`B-paint-layer-destroys-other-layer-weights` (both quote `FinalWeightBlending` only as the name of
the merge's shader pass, neither claims a layer is not in it),
`B-configure-layer-blend-wrong-nodes`, `E-material-configure-layer-blend-blendtype-undiscoverable`
and `B-material-authoring-save-no-disk-write` (all about the material graph or the asset save, not
the blend method value). **Nothing on the board owns the claim that the auto-created LayerInfo is
non-weight-blended, or that the engine factory and its `DefaultLayerInfoObject` hook are
bypassed.**

## Not done

No source modified, no test written, no editor call made (pid 18592 untouched — this is a source
re-derivation over already-measured data). The proposed `warnings[]` line and response field are
not prototyped. Only this project was checked for the settings override; the `None` default is the
engine's and applies anywhere `TargetLayerDefaultBlendMethod` is unset, but a project that sets it
would not see this.

severity rationale: impact=High — the README's silent-wrong-data band, *"the caller trusts a result that is a lie and builds on it"*: the verb's own tool description states in capitals that painting at strength 1.0 "drives every other layer in that blend group toward zero WITHIN the painted region" and sells the verify census as making "the cost to the other layers a reported number instead of a discovery in Landscape Ed Mode" (`LandscapeHandler.cpp:2463`), while the layer it manufactures is flagged `No Weight Blending` and is excluded from the normalisation pass at `LandscapeEditLayers.cpp:3594`; the caller builds multi-layer terrain on a displacement that never happens, gets `success: true` with a clean census, and no field in the response mentions a blend method × reach=normal — **the modifier is declined in both directions and both readings are named**. No bump up: layer painting is terrain authoring, not an almost-every-session activity — the same reading `B-layer-paint-doc-claims-blend-group-write` § *Severity* took for the same verb, kept identical here on purpose so the two sort predictably. No bump down: the affected path is not a rare edge, it is the **default** one — the auto-create branch is what runs for every layer that has no LayerInfo asset yet, i.e. the first paint of every layer on any landscape authored through this verb (`layerInfoAutoCreated: true`, seen on three of four paints in the run cited above), and every layer created that way stays non-blended for the life of the landscape. Critical is declined on the band's own words, *"a write that corrupts or loses asset data"*: nothing is corrupted and nothing is lost — the weightmaps hold precisely the bytes that were written, and the state is repairable in place by setting Blend Method on the LayerInfo. Rating it Critical would order it above `B-paint-erases-orphaned-layer`'s genuinely irreversible landscape-wide erase, which is the wrong order for the picker. Medium is declined because Medium is a soft blocker reachable "via a documented workaround, a source dive, or many extra calls" — there is no workaround at all through this verb: the parameter does not exist, the response does not report the state, the docs assert the opposite, and the engine's own project-level cure (`DefaultLayerInfoObject`) is bypassed by the raw `NewObject`, so no configuration the caller can make changes the outcome -> High

## History
- `#1-auto-created-layerinfo-is-blendmethod-none` `OPEN` reporter — **`landscape.create_procedural_terrain`'s auto-created `ULandscapeLayerInfoObject` is `ELandscapeTargetLayerBlendMethod::None` — display name "No Weight Blending" — so the group normalisation its own registered summary promises in capitals cannot occur, on the default path, silently.** Re-derived at plugin HEAD `1a9e5778` and UE 5.8; every citation opened and read for this filing, none relayed. **The chain.** The normalisation is the merge's `FLandscapeEditLayersWeightmapsPerformFinalWeightBlendingPS` (`LandscapeEditLayers.cpp:791`), and a layer is admitted to it only by `if (InLayerInfo->GetBlendMethod() == ELandscapeTargetLayerBlendMethod::FinalWeightBlending)` at `:3594`. A freshly constructed LayerInfo takes its method from project settings in its own constructor — `BlendMethod = GetDefault<ULandscapeSettings>()->GetTargetLayerDefaultBlendMethod();` (`LandscapeLayerInfoObject.cpp:27`, accessor `LandscapeSettings.h:87`) — whose compiled default is `ELandscapeTargetLayerBlendMethod::None` (`LandscapeSettings.h:167`), and `None = 0 UMETA(DisplayName = "No Weight Blending", Tooltip = "The target layer's weight is unaffected by other target layers.")` (`LandscapeEditTypes.h:34`). **Checked, not assumed:** the class is `UCLASS(config = Engine, defaultconfig)` (`LandscapeSettings.h:45`), and grepping this project's `Config/`, its `Saved/Config/` and `C:/UE_5.8/Engine/Config/` for both `TargetLayerDefaultBlendMethod` and `[/Script/Landscape.LandscapeSettings]` returns zero hits in all three. **Nothing in PinWright sets it:** `LandscapeHandler.cpp` has zero `BlendMethod` references and the only one in all of `Source/` is `MaterialAuthoringHandler.cpp:3211`, a different verb behind an explicit parameter. The auto-create branch (`:2777-2825`) does `NewObject` + `SetLayerName` + `CreateTargetLayerSettingsFor`, and that last call only registers — it loops proxies calling `AddTargetLayer`/`UpdateTargetLayer` (`Landscape.cpp:4478-4493`) and touches no blend method. **The half that makes this a plugin defect rather than an unhelpful engine default:** the engine's single blessed factory `UE::Landscape::CreateTargetLayerInfo` (`LandscapeUtils.cpp:286`, used by the Target Layers panel `..._TargetLayers.cpp:2354`, import layers `..._ImportLayers.cpp:414`, `LandscapeEditorObject.cpp:962`, `WorldTileCollectionModel.cpp:1564`) begins by loading `Settings->GetDefaultLayerInfoObject()` and **duplicating** it when set (`LandscapeUtils.cpp:288-302`) — that soft pointer (`LandscapeSettings.h:128`, getter `:77`) is the project-level knob the `TargetLayerDefaultBlendMethod` doc comment itself defers to. PinWright calls raw `NewObject` at `LandscapeHandler.cpp:2778-2780` and `CreateTargetLayerInfo` appears nowhere in `Source/`, **so a project that configures `DefaultLayerInfoObject` to a weight-blended template still gets a `None` layer out of this verb** — the one control that exists is disconnected from the path that needs it. **5.7-era flip, bounded honestly:** `PostLoad` migrates pre-`LandscapeAdvancedWeightBlending` assets as `bNoWeightBlend_DEPRECATED ? None : FinalWeightBlending` (`LandscapeLayerInfoObject.cpp:276`, contract at `LandscapeLayerInfoObject.h:80-82`), so pre-5.7 assets stay blended and only newly constructed ones land on the settings default; whether the pre-5.7 constructor default was blended is NOT established (that source is not in this tree) and is not claimed. **Measurement attributed, not re-run:** the discriminator is `#5` on `B-paint-layer-destroys-other-layer-weights` (board commit `9eed329`), read in full — its paint (3) read `texelsAtRequestedWeight: 256` for `Sand` (all 256 texels at exactly 255) while `layersAffected[]` showed `Grass` and `Rock` still holding weight in the same texels, a per-texel sum a normalised weightmap cannot hold. Running editor is the 13:32 build, plugin `d8f1bc32`, and `git diff --name-only d8f1bc32..HEAD -- '*.cpp' '*.h'` is empty, so that run and these citations are the same code. No editor call was made for this filing. **Fix:** route the auto-create through `UE::Landscape::CreateTargetLayerInfo` (reconnecting `DefaultLayerInfoObject`), or at minimum call `SetBlendMethod(FinalWeightBlending)` as `MaterialAuthoringHandler.cpp:3211` already does, with an opt-out parameter; publish the blend method per layer in `layersAffected[]`; and warn when a painted layer is not `FinalWeightBlending`. **Rider folded in, Low on its own:** `LandscapeHandler.cpp:2818-2819` calls `CreateLayerEditorSettingsFor` "a deprecated empty stub" on 5.5+, but a recursive grep over `C:/UE_5.8/Engine/Source/` finds the symbol nowhere — removed outright, not stubbed. Harmless behind the pre-5.5 `#if`, misleading to a backporter, and folded here rather than ticketed because it sits inside the exact branch this fix rewrites. **Dedup:** board-wide grep for `BlendMethod`, `FinalWeightBlending`, `noWeightBlend`, `blend method`, `weight-blend`, `add_landscape_layer` with every hit's frontmatter read; the five matching tickets are about the merge shader's *name*, the material graph nodes, or the asset save, and none owns the claim that the auto-created LayerInfo is non-blended or that the engine factory is bypassed. **Carved deliberately from two neighbours filed today, with the split test stated:** the doc half stays on `B-layer-paint-doc-claims-blend-group-write` (its `#2` carries this root cause and points here) because neither fix closes the other, and the sibling verb's inverted default is `B-add-landscape-layer-noweightblend-inverted-default` because routing this verb through the factory does not touch that handler. `B-paint-layer-destroys-other-layer-weights` is cited and deliberately **not** edited; its settled verdict is not reopened here. **Severity `High`** on the silent-wrong-data band with Critical and Medium both argued down in the rationale line, and the reach modifier declined in both directions — no bump up because layer painting is not an every-session activity, no bump down because the auto-create branch is the default path for the first paint of every layer, not an edge.
