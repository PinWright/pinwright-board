---
id: E-material-configure-layer-blend-blendtype-undiscoverable
title: "material.authoring.configure_layer_blend wiki page enumerates no layers[].blendType values — the value is discoverable only by knowing UE's ELandscapeLayerBlendType, and a wiki grep returns a misleading wrong-namespace hit"
status: OPEN
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
