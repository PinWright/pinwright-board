---
id: B-add-landscape-layer-noweightblend-inverted-default
title: "material.authoring.add_landscape_layer only calls SetBlendMethod when noWeightBlend is explicitly present, so omitting a parameter named \"Disable weight blending\" is what disables weight blending — the created asset lands on the constructor default, ELandscapeTargetLayerBlendMethod::None"
status: OPEN
severity: Medium
category: bug
tags: [material-authoring, add_landscape_layer, landscape, blend-method, weight-blending, layerinfo, inverted-default, parameter-semantics, ue57-regression, landscape-settings]
encounters: 1
lastSeen: 2026-08-30T16:50:00+03:00
---

# The opt-out parameter is the only thing that can opt you in

`material.authoring.add_landscape_layer` creates a shared `ULandscapeLayerInfoObject` asset. It
takes an optional boolean whose entire documented meaning is *"Disable weight blending"*
(`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:3157`). A caller reading
that name expects the obvious contract: pass it to turn weight blending **off**, omit it and the
layer is weight-blended, which is what a landscape target layer is normally for.

The handler does the opposite, by omission:

```cpp
bool bNoWeightBlend = false;
if (Payload.IsValid() && Payload->TryGetBoolField(TEXT("noWeightBlend"), bNoWeightBlend))
{
#if UE_VERSION_NEWER_THAN_OR_EQUAL(5, 7, 0)
    LayerInfo->SetBlendMethod(bNoWeightBlend ? ELandscapeTargetLayerBlendMethod::None
                                             : ELandscapeTargetLayerBlendMethod::FinalWeightBlending, false);
#else
    LayerInfo->bNoWeightBlend = bNoWeightBlend;
#endif
}
// MaterialAuthoringHandler.cpp:3207-3214
```

The `SetBlendMethod` call is **inside** the `TryGetBoolField` guard. Omit the field and no blend
method is ever set, so the asset keeps whatever its constructor gave it — and on 5.7+ that is
`ELandscapeTargetLayerBlendMethod::None`, whose engine display name is literally **"No Weight
Blending"**.

So the three reachable states are:

| call | resulting `BlendMethod` |
|---|---|
| `noWeightBlend: true` | `None` — no weight blending |
| `noWeightBlend: false` | `FinalWeightBlending` |
| *omitted* | **`None` — no weight blending** |

Omitting the disable flag and passing it as `true` produce the identical asset. The only way to
get a weight-blended layer out of this verb is to explicitly pass the *disable* flag as `false`.

## Re-derived at plugin HEAD `1a9e5778` and UE 5.8

The constructor default and its provenance, read for this ticket rather than relayed:

```cpp
// The default blend method comes from the settings :
BlendMethod = GetDefault<ULandscapeSettings>()->GetTargetLayerDefaultBlendMethod();
// C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeLayerInfoObject.cpp:27
```

```cpp
UPROPERTY(Config, Category = "Target Layers", EditAnywhere)
ELandscapeTargetLayerBlendMethod TargetLayerDefaultBlendMethod = ELandscapeTargetLayerBlendMethod::None;
// Runtime/Landscape/Public/LandscapeSettings.h:167   (accessor :87, UCLASS config=Engine at :45)
```

```cpp
None = 0 UMETA(DisplayName = "No Weight Blending",
               Tooltip = "The target layer's weight is unaffected by other target layers."),
// Runtime/Landscape/Public/LandscapeEditTypes.h:34
```

**Confirmed unset rather than assumed:** grepping this project's `Config/`, its `Saved/Config/`
and `C:/UE_5.8/Engine/Config/` for both `TargetLayerDefaultBlendMethod` and the
`[/Script/Landscape.LandscapeSettings]` section returns **zero hits in all three**.

The asset is constructed with a bare `NewObject<ULandscapeLayerInfoObject>` at
`MaterialAuthoringHandler.cpp:3174-3175`, not through the engine's factory
`UE::Landscape::CreateTargetLayerInfo` (`Runtime/Landscape/Private/LandscapeUtils.cpp:286`), so the
project-level `DefaultLayerInfoObject` template hook (`LandscapeSettings.h:128`, getter `:77`,
duplicated at `LandscapeUtils.cpp:298-302`) is bypassed here too — a project that configures it
still gets `None` from this verb. That is the same structural gap as the paint verb's, and it is
the reason the two tickets share a fix idea even though they do not share a seat.

## Why this reads as a 5.7 regression, and the limit of that claim

The `#else` branch writes `bNoWeightBlend` directly on pre-5.7, and the deprecation contract
records that in the legacy model `false` meant weight-blended:

```cpp
UE_DEPRECATED(5.7, "bNoWeightBlend has been replaced by BlendMethod (false is
    ELandscapeTargetLayerBlendMethod::FinalWeightBlending, true is ELandscapeTargetLayerBlendMethod::None)")
// Runtime/Landscape/Classes/LandscapeLayerInfoObject.h:80-82
```

and `PostLoad` migrates old assets by exactly that mapping (`LandscapeLayerInfoObject.cpp:276`).

**What that does NOT establish, stated rather than glossed:** whether the pre-5.7 *constructor*
default for `bNoWeightBlend` was `false`. That source is not in this tree. If it was, this verb's
omission path was harmless before 5.7 and the guard only became wrong when the default moved into
project settings — plausible, and not asserted.

## Impact

The asset is real, shared, and usually created precisely so a landscape can weight-blend the layer
— `configure_layer_blend` is documented as consuming these (`B-configure-layer-blend-wrong-nodes`
quotes its summary: *"Used by configure_layer_blend to set up weight-blended layers"*). A caller
who never heard of `noWeightBlend` gets an asset that is silently excluded from the merge's final
weight-blending pass (`LandscapeEditLayers.cpp:791`, gated at `:3594`), so weights across the blend
group stop summing to 1 and painting one layer no longer displaces its siblings.

The response does not report the blend method (`AddAssetVerification` + `layerName`,
`MaterialAuthoringHandler.cpp:3225-3227`), so nothing at the call site reveals it.

## Fix

Set the blend method unconditionally, defaulting to weight-blended:

```cpp
const bool bNoWeightBlend = Ctx.GetBool(TEXT("noWeightBlend"), false);
LayerInfo->SetBlendMethod(bNoWeightBlend ? ELandscapeTargetLayerBlendMethod::None
                                         : ELandscapeTargetLayerBlendMethod::FinalWeightBlending, false);
```

— i.e. move the call out of the `TryGetBoolField` guard and read the flag with an explicit default,
which is how every other optional boolean in this handler is read. Then **echo the resulting blend
method in the response** so the caller can see which of the three states they got.

Two things worth deciding rather than defaulting:

- **`PremultipliedAlphaBlending` is unreachable.** The enum has three usable values
  (`LandscapeEditTypes.h:34-36`); a boolean can only select two, and the one it cannot reach is the
  engine's *"Advanced Weight Blending"*, described there as the method that works correctly with
  edit layers — while `FinalWeightBlending` is labelled *"Weight Blending (Legacy)"* with the
  tooltip *"Doesn't work well when combined with edit layers"*. Since this plugin paints through
  edit layers, a `blendMethod: "None" | "FinalWeightBlending" | "PremultipliedAlphaBlending"`
  string parameter — with `noWeightBlend` kept as a deprecated alias — is the better shape than
  fixing the boolean's default. Filed as part of this ticket rather than a separate `F-` because
  the boolean's default has to be touched either way and the enum is the same edit.
- Consider routing through `UE::Landscape::CreateTargetLayerInfo` so `DefaultLayerInfoObject` works
  — shared with `B-paint-auto-created-layer-never-weight-blended`, and cheap to do once for both.

## Distinct from

- **`B-paint-auto-created-layer-never-weight-blended`** (OPEN, High) — the same root *value*
  (`None`) on a different verb, different namespace, different file, **and neither fix closes the
  other**: routing `landscape.create_procedural_terrain`'s auto-create through the engine factory
  does not touch this handler, and moving this `SetBlendMethod` out of its guard does not touch
  that branch. That independence is the split test, and it is why these were filed as two rather
  than one ticket spanning both verbs — the same non-bundling argument
  `B-layer-paint-doc-claims-blend-group-write` § *Structure* makes. They rate differently too,
  which is the second reason: there the verb's own tool description asserts the normalisation in
  capitals and the caller is actively misinformed; here the summary claims nothing about blending
  and the caller is merely uninformed.
- **`B-material-authoring-save-no-disk-write`** (IN-REVIEW, High) — same verb, and its `#3`
  documents that this creation path ends in a bare `MarkPackageDirty()` and never reaches disk.
  Different axis entirely: that ticket is about the asset not persisting, this one about what is
  in it. Its citations are stale (`:2509`, `:2576-2582`); the file has since grown and the
  creation now sits at `:3150-3229`. Worth a fixer's attention because **both defects are in the
  same 70 lines** and one edit pass could sensibly close both — sequencing, not merging.
- **`B-configure-layer-blend-wrong-nodes`** (OPEN) — the material-graph half of "make a layer
  paintable". Complementary and independent: that ticket is about the `LandscapeLayerBlend` node
  that *declares* a target layer, this one about the `ULandscapeLayerInfoObject` that binds it.
- **`E-material-configure-layer-blend-blendtype-undiscoverable`** (OPEN) — about the
  `LandscapeLayerBlend` node's per-layer `BlendType` (`LB_WeightBlend` / `LB_AlphaBlend`), which is
  a *material graph* property. Named because "blend type" dedups onto it and it is a genuinely
  different property from `ULandscapeLayerInfoObject::BlendMethod`; a reader should not conflate
  them.

## Same shape as

Not a member of the session's recurring class stated on `B-foliage-paint-does-no-ground-projection`
§ *Same shape as* (*the call succeeds, every number it reports is correct, and the output is wrong
because the deciding number was never reported*). **Declined with a reason:** this verb reports no
number that could be over-read — it returns an asset path and a layer name — and it makes no
promise about blending in its summary, so nothing it says is a lie. The defect is a parameter whose
omission selects the state its own name describes as the exception. The nearer neighbour is
`B-add-variable-default-value-ignored` — an optional input whose absence yields a state the caller
would not choose — not the silent-wrong-number class.

## Dedup

Board-wide grep for `noWeightBlend`, `BlendMethod`, `FinalWeightBlending`, `blend method`,
`weight-blend` and `add_landscape_layer`, with every hit's frontmatter read. Seven files mention
`add_landscape_layer`; the ones with any blend content are
`B-configure-layer-blend-wrong-nodes`, `E-material-configure-layer-blend-blendtype-undiscoverable`
and `B-material-authoring-save-no-disk-write`, all carved out above. `B-layer-paint-doc-claims-blend-group-write`
and `B-paint-layer-destroys-other-layer-weights` name `FinalWeightBlending` only as the merge
shader's name. **Nothing on the board mentions the `noWeightBlend` guard or the inverted omission
default.**

## Not done

No source modified, no test written, no editor call made (pid 18592 untouched). The three-state
table above is read off `MaterialAuthoringHandler.cpp:3207-3214` plus
`LandscapeLayerInfoObject.cpp:27` and was **not** confirmed by creating an asset and inspecting it;
the code path admits no other outcome, but it is a derivation, not a measurement, and is labelled
so. The pre-5.7 constructor default is unverified, as stated above.

severity rationale: impact=Medium — the README's soft-blocker band, *"doable, but only via a documented workaround, a source dive, or many extra calls"*: a weight-blended layer info **is** reachable through this verb today by passing `noWeightBlend: false` explicitly, which is a real one-argument workaround, and the created asset is a normal `/Game` asset whose Blend Method is visible in the details panel — so a caller who suspects something can both check it and fix it without leaving the tool; what is missing is any reason to suspect, since the parameter's name implies the opposite and the response does not echo the value × reach=normal — **both modifiers declined and named**. No bump up: `material.authoring.add_landscape_layer` is an infrequent authoring verb, not an almost-every-session one. No bump down: the affected path is the *default* one — every call that omits an optional parameter — not a rare edge, so the rubric's edge-path clause does not apply; how often the verb itself is called is an `encounters` signal, which the README excludes as a severity input. High is declined because the High band is silent wrong data *on the wire*, and this verb asserts nothing about blending: its registered summary (`MaterialAuthoringHandler.cpp:3150-3151`) says only that it creates a shared LayerInfo and explicitly that it does NOT make a layer paintable, and its response carries an asset path and a layer name, neither of which is false. That is precisely the line that separates this from `B-paint-auto-created-layer-never-weight-blended`, where the tool description states the group normalisation in capitals and the caller is actively misinformed — the two are deliberately rated a band apart so the picker takes the misinforming one first. Low is declined because the Low band is *"docs, discoverability, naming, ... or cosmetic"*, and a fix limited to renaming the parameter or documenting the trap would leave every existing caller's assets non-blended: the wrong value is written into a persisted asset, so this is behavioural, not documentary -> Medium

## History
- `#1-omitting-the-disable-flag-disables-blending` `OPEN` reporter — **`material.authoring.add_landscape_layer` calls `SetBlendMethod` only from inside `if (Payload->TryGetBoolField(TEXT("noWeightBlend"), bNoWeightBlend))` (`MaterialAuthoringHandler.cpp:3207-3214`), so omitting a parameter documented as "Disable weight blending" (`:3157`) leaves the asset on its constructor default — `ELandscapeTargetLayerBlendMethod::None`, engine display name "No Weight Blending". Omitting the flag and passing it `true` produce the identical asset; the only way to get a weight-blended layer is to explicitly pass the *disable* flag as `false`.** Re-derived at plugin HEAD `1a9e5778` and UE 5.8, every citation opened for this filing. The asset is a bare `NewObject<ULandscapeLayerInfoObject>` (`:3174-3175`); its constructor sets `BlendMethod = GetDefault<ULandscapeSettings>()->GetTargetLayerDefaultBlendMethod()` (`LandscapeLayerInfoObject.cpp:27`, accessor `LandscapeSettings.h:87`), whose compiled default is `None` (`LandscapeSettings.h:167`), and `None = 0 UMETA(DisplayName = "No Weight Blending", Tooltip = "The target layer's weight is unaffected by other target layers.")` (`LandscapeEditTypes.h:34`). **Checked rather than assumed:** the class is `UCLASS(config = Engine, defaultconfig)` (`:45`) and grepping this project's `Config/`, its `Saved/Config/` and `C:/UE_5.8/Engine/Config/` for both `TargetLayerDefaultBlendMethod` and `[/Script/Landscape.LandscapeSettings]` returns zero hits in all three. **Consequence:** the layer is excluded from the merge's final weight-blending pass (`LandscapeEditLayers.cpp:791`, gated at `:3594`), so weights across the blend group stop summing to 1 — and the response reports only an asset path and layer name (`:3225-3227`), never the blend method, so nothing at the call site reveals it. **Reads as a 5.7 regression, with the limit stated:** the `#else` branch writes `bNoWeightBlend` directly on pre-5.7, and the deprecation contract records `false -> FinalWeightBlending` (`LandscapeLayerInfoObject.h:80-82`, migration at `LandscapeLayerInfoObject.cpp:276`) — but whether the pre-5.7 *constructor* default was `false` is NOT established (that source is not in this tree) and is not claimed. **The engine factory is bypassed here too:** `UE::Landscape::CreateTargetLayerInfo` (`LandscapeUtils.cpp:286`) would duplicate a configured `DefaultLayerInfoObject` (`LandscapeSettings.h:128`, `LandscapeUtils.cpp:298-302`); this verb's raw `NewObject` cannot, so the project-level cure does not reach it. **Fix:** move the call out of the guard and read the flag with an explicit default (`Ctx.GetBool(TEXT("noWeightBlend"), false)`), as every other optional boolean in this handler is read, and echo the resulting blend method in the response. Recommended in the same edit: replace the boolean with a `blendMethod` string, because a boolean cannot reach `PremultipliedAlphaBlending` — the engine's *"Advanced Weight Blending"*, which its own tooltip says is the one that works with edit layers, while `FinalWeightBlending` is labelled *"Weight Blending (Legacy)"* and *"Doesn't work well when combined with edit layers"* (`LandscapeEditTypes.h:34-36`), and this plugin paints through edit layers. **Filed separately from `B-paint-auto-created-layer-never-weight-blended` (High), and the split test is stated:** same root value, different verb, namespace, file and seat, and **neither fix closes the other** — routing the paint verb's auto-create through the engine factory does not touch this handler, and moving this `SetBlendMethod` out of its guard does not touch that branch. They also rate a band apart on purpose: there the tool description asserts the normalisation in capitals so the caller is actively misinformed; here the summary claims nothing about blending and the caller is merely uninformed. **Dedup:** board-wide grep for `noWeightBlend`, `BlendMethod`, `FinalWeightBlending`, `blend method`, `weight-blend`, `add_landscape_layer` with every hit's frontmatter read — `B-configure-layer-blend-wrong-nodes` (material graph nodes), `E-material-configure-layer-blend-blendtype-undiscoverable` (the `LandscapeLayerBlend` node's `BlendType`, a different property that should not be conflated with this one) and `B-material-authoring-save-no-disk-write` (`#3`, same verb, same 70 lines, but about the asset never reaching disk — sequence with it, do not merge; its citations `:2509`/`:2576-2582` are stale, the creation now sits at `:3150-3229`) are the only related hits, and nothing owns the guard or the inverted omission default. **Severity `Medium`** on the soft-blocker band — `noWeightBlend: false` is a real one-argument workaround and Blend Method is visible on the created asset — with High declined (this verb asserts nothing false on the wire, unlike its High-rated sibling) and Low declined (a wrong value is persisted into an asset; renaming or documenting the parameter would not repair it). Reach declined both ways: infrequent verb, but the affected path is the default one, and verb popularity is an `encounters` signal the README excludes from severity. **Not done:** no source modified, no test, no editor call; the three-state table is derived from `:3207-3214` plus the constructor, not confirmed by creating an asset and inspecting it.
