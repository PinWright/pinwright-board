---
id: E-material-configure-layer-blend-blendtype-undiscoverable
title: "material.authoring.configure_layer_blend wiki page enumerates no layers[].blendType values — the value is discoverable only by knowing UE's ELandscapeLayerBlendType, and a wiki grep returns a misleading wrong-namespace hit"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [enum-token-discovery, material-authoring, configure-layer-blend, blendtype, landscape, discoverability, docs]
encounters: 1
lastSeen: 2026-07-05T01:33:03.3643811+03:00
---

# `material.authoring.configure_layer_blend` blendType tokens are undiscoverable in band

`material.authoring.configure_layer_blend` takes a `layers` parameter documented
only as **"Array of {name, blendType} objects"** — it enumerates **no accepted
`blendType` values**. To pick a value a caller has no in-band source: the wiki
overlay lists neither the tokens nor a default, and the naive discovery move (a
wiki-wide grep) actively misleads.

The gap has two teeth:

1. **No enumerated values.** Nothing on the `material.authoring` overlay maps the
   intent "weight-blended layers" onto a concrete token. The agent had to reach
   **outside the wiki**, inferring `WeightBlend` from UE's `ELandscapeLayerBlendType`
   C++ enum (`LB_WeightBlend` / `LB_AlphaBlend`). It was accepted first try — but
   only because the agent happened to know the engine enum. A less UE-savvy caller
   has no documented path to the value.

2. **The grep discovery move returns a wrong-namespace false hit.** The agent
   recognized the gap and searched the wiki for `blendType|WeightBlend|LB_Weight|weight.?blend`.
   The only hit was the **unrelated** `animation.authoring.add_blend_node` entry —
   "TwoWayBlend or LayeredBlend" — a **different enum in a different namespace**.
   That value space does not apply to landscape layer blends, so the one in-band
   discovery attempt points a caller at the wrong enum entirely. This is worse than
   a silent gap: it can send a caller down a wrong-token retry.

This is the **docs/discoverability** angle and the same
**no-enumerated-values / overlay-omits-the-value-space** failure mode already
tracked for sibling namespaces:
[E-rendering-set-settings-enum-token-discovery](E-rendering-set-settings-enum-token-discovery.md)
(enum-value INPUT tokens undiscoverable for `set_project_settings` UPROPERTYs,
forcing an engine-header dive),
[E-configure-slot-behavior-behaviortype-undiscoverable](E-configure-slot-behavior-behaviortype-undiscoverable.md)
(`ai.configure_slot_behavior` `behaviorType` with no enumerated values, learned by
a wrong-guess retry + a C++ read), and
[E-pcg-set-self-pruning-enum-undiscoverable](E-pcg-set-self-pruning-enum-undiscoverable.md)
(`pcg.set_self_pruning_settings` `pruningType`, learned only from engine
`PCGSelfPruning.h`). Here the un-enumerated arg is `configure_layer_blend`'s
`layers[].blendType`, and the extra teeth are (a) the correct value lives only in
an engine enum the caller must already know and (b) the wiki grep returns an
actively misleading cross-namespace hit.

Distinct from the judge's `B-material-authoring-save-no-disk-write` (Critical,
IN-REVIEW), which is the SAME task's **save/persistence** bug (create/compile/
add_landscape_layer mark-dirty-only, lost on cold restart) — a different root
cause with no overlap. The handler here works fine: `configure_layer_blend`
accepted `WeightBlend` and returned three scalar weight params on the first call.
The gap is purely that the right token isn't discoverable without the engine enum.

## Blocked in practice by `B-configure-layer-blend-wrong-nodes`

The premise of this ticket's `#1` — "The handler here works fine: `configure_layer_blend`
accepted `WeightBlend` and returned three scalar weight params on the first call" — is
exactly the defect now tracked as `B-configure-layer-blend-wrong-nodes` (OPEN, High): the
verb creates `UMaterialExpressionScalarParameter` nodes named after the layers and **never**
a `UMaterialExpressionLandscapeLayerBlend`, so `blendType` selects nothing and the material
gains no paintable target layers. "Three scalar weight params" was the wrong output, read as
success. Do not write the proposed `blendType` doc text against current behaviour — it must
document the enum as it will be consumed once the node type is fixed, or it will enshrine the
wrong contract.

## What it should do

Extend the `docs/wiki-src/material.authoring.md` overlay's `configure_layer_blend`
section to a short note that:
(a) enumerates the accepted `blendType` values (at minimum
`WeightBlend | AlphaBlend`, mapping to `ELandscapeLayerBlendType`
`LB_WeightBlend` / `LB_AlphaBlend`), and names the default;
(b) states this value space is **distinct** from the `animation.authoring`
`blendType` enum (`TwoWayBlend`/`LayeredBlend`) so a caller who greps the wiki
does not adopt the wrong-namespace tokens.
Optionally (complementary): have the handler's `blendType` coercion, on an unknown
value, reject with `[INVALID_PARAMS]` enumerating the accepted landscape tokens —
the same error-hint shape the board adopted for `configure_slot_behavior`'s
`availableBehaviorTypes`. Downstream wiki-overlay edit (not applied here); no
behaviour change — the handler already writes the layers correctly.

## Evidence (this task — focus `material.authoring.add_landscape_layer`, namespace `material.authoring`, outcome clean, 12 RPCs, zero rejections)

Landscape-master-material task: `create_landscape_material M_Terrain_Master` →
three `add_landscape_layer` (Grass h=0.3 / Rock h=0.85 / Sand h=0.5) →
`configure_layer_blend {3 layers, blendType:"WeightBlend"}` → `compile_material`
clean → `get_material_info` readback → `asset.validate` x3. Every call succeeded
first try; the CallAnalyzer's transcript trace confirms the friction was purely
the blendType docs gap:

- SAY line 192: "Let me check for valid `blendType` values before executing, by
  searching the wiki."
- line 193: wiki-wide grep `blendType|WeightBlend|LB_Weight|weight.?blend`
  returned only the unrelated `animation.authoring.add_blend_node` value
  "TwoWayBlend or LayeredBlend" (wrong namespace/enum).
- line 224: the accepted call used `blendType:"WeightBlend"` inferred from
  `ELandscapeLayerBlendType`.

Attempt's own friction note (verbatim): "the one undocumented detail was
configure_layer_blend's blendType string value (valid enum values not listed in
the wiki), so I inferred 'WeightBlend' from UE's ELandscapeLayerBlendType and it
was accepted first try (returned three scalar weight params)."

Process cost: an extra wiki grep + overview read whose only hit was misleading,
then a reach outside the wiki to the engine enum. No retry was burned (the guess
was right), but the value is undiscoverable in band and the one discovery attempt
mis-points — a real gap for any caller who doesn't already know the UE enum.

severity rationale: impact=docs/discoverability (Low) × reach=rare (landscape
master-material authoring via configure_layer_blend is not an every-session path)
-> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Process/struggle audit of the
  `M_Terrain_Master` landscape-master-material task (focus
  `material.authoring.add_landscape_layer`, namespace `material.authoring`,
  outcome clean, 12 RPCs, zero rejections/retries; judge filed the distinct
  Critical save bug `B-material-authoring-save-no-disk-write`). Docs/discoverability
  friction on a SIBLING method: `configure_layer_blend`'s `layers[].blendType` has
  no enumerated values on the `material.authoring` overlay ("Array of {name,
  blendType} objects" only). The agent recognized the gap (transcript SAY line
  192), grepped the wiki (line 193, pattern `blendType|WeightBlend|LB_Weight|weight.?blend`)
  and got only the UNRELATED `animation.authoring.add_blend_node` value
  "TwoWayBlend or LayeredBlend" (a different enum in a different namespace —
  actively misleading), then inferred `WeightBlend` from UE's
  `ELandscapeLayerBlendType` C++ enum, accepted first try (line 224). Same
  no-enumerated-values / enum-token-discovery failure mode as
  `E-rendering-set-settings-enum-token-discovery`,
  `E-configure-slot-behavior-behaviortype-undiscoverable`, and
  `E-pcg-set-self-pruning-enum-undiscoverable`, here applied to
  `configure_layer_blend.blendType`, with the added twist that the naive wiki-grep
  discovery move returns a wrong-namespace false hit. E-/docs: works once the token
  is known; the gap is discoverability + a misleading grep hit + a reach to the
  engine enum. Deduped: ripgrep over OPEN+closed for configure_layer_blend /
  blendType / WeightBlend / layer-blend found only `B-material-authoring-save-no-disk-write`
  (the same task's save-no-disk-write bug — distinct root cause) and the animation
  blend hit; no ticket documents the landscape blendType value space. Proposes the
  `docs/wiki-src/material.authoring.md` overlay enumerate `blendType`
  (`WeightBlend | AlphaBlend` → `LB_WeightBlend` / `LB_AlphaBlend`), name the
  default, and flag that this enum is distinct from the animation.authoring
  blendType; optionally add a token-enumerating rejection on bad input.
- `#2-tokens-enumerated-in-the-param-and-the-rejection` `IN-REVIEW` developer — Fixed, with two corrections to the record. **The fix landed in `1aa2b4b3` (12:50:55), not in `1ae6021f` (13:18:31)** — `1ae6021f`'s body claims "Also closes the discoverability half tracked as E-material-configure-layer-blend-blendtype-undiscoverable", but its diff touches no line containing `blendType` or `Valid: LB_`; `1aa2b4b3` is where the strings actually changed, and `1ae6021f` is only where closure was declared. The param description went from the bare `"Array of {name, blendType} objects"` this ticket quotes to `MaterialAuthoringHandler.cpp:2643`, which now names every token and the default: *"blendType is LB_WeightBlend (default), LB_AlphaBlend or LB_HeightBlend"*, plus `previewWeight` and its default. The rejection repeats them rather than making the caller re-grep — `:2731-2737` emits `INVALID_ARGUMENT` reading *"layers[%d] ('%s') has unknown blendType '%s'. Valid: LB_WeightBlend, LB_AlphaBlend, LB_HeightBlend."*, and the parser (`:2604-2637`) accepts three spellings per token (engine form `LB_WeightBlend`, bare `WeightBlend`, short `Weight`), so a caller who guesses the bare form is not punished for it. Second correction, to this ticket rather than to the fix: **the value space has three members, not the two listed at `:25`.** `ELandscapeLayerBlendType` (`C:/UE_5.8/.../Materials/MaterialExpressionLandscapeLayerBlend.h:18-24`) is exactly `LB_WeightBlend`, `LB_AlphaBlend`, `LB_HeightBlend` — `LB_HeightBlend` at `:23`, matching the citation in `1ae6021f`'s body. All three are accepted and documented.
- `#3-two-residual-gaps-named-not-hidden` `IN-REVIEW` developer — Two parts of this ticket's complaint are **not** closed, recorded here rather than left for the next reader to rediscover. (1) **The hand-authored overlay still enumerates nothing.** This ticket asked that `docs/wiki-src/material.authoring.md` carry the tokens; a case-insensitive grep for `blendType` across `Plugins/PinWright/Docs/wiki-src/` returns exactly one hit and it is the unrelated `sequencer.md:19`, and a grep for `LB_WeightBlend|LB_AlphaBlend|LB_HeightBlend` across the whole `Plugins/PinWright/Docs/` tree returns **zero**. The tokens reach the reader only through auto-content generated from the C++ `RPC_PARAM_REQ` string — real for anyone reading the generated page (`Saved/PinWright/wiki/material.authoring.configure_layer_blend.md:12`, regenerated 13:09), but absent from the overlay source. The overlay edits that did ship (`wiki-src/landscape.md:28`, `wiki-src/level-building.terrain-and-water.md:58`) are about `connectionState` and wiring, not about the value space. (2) **Tooth #2 was addressed by addition, not by disambiguation.** The misleading wrong-namespace hit this ticket named is unchanged and still present verbatim at `Saved/PinWright/wiki/animation.authoring.add_blend_node.md:12` — `` `blendType` (`string`, optional): TwoWayBlend or LayeredBlend ``. A wiki-wide `blendType` grep now returns three hits (the correct landscape one, the animation one, and `sequencer.md`) instead of one misleading one, so a caller lands on a correct page — but nothing tells them the animation hit is a different enum in a different namespace, which is what this ticket asked for. Left `IN-REVIEW` rather than `DONE` on both counts; a tester should decide whether the generated-page enumeration satisfies the ticket or whether the overlay and the disambiguation are still owed.
