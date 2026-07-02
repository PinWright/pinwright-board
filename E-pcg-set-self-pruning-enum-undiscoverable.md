---
id: E-pcg-set-self-pruning-enum-undiscoverable
title: "pcg.set_self_pruning_settings wiki page enumerates no pruningType values and doesn't state RadiusSimilarityFactor only applies to the radius modes — discoverable only by reading engine PCGSelfPruning.h"
status: OPEN
severity: Low
category: ergonomic
tags: [pcg, set-self-pruning-settings, pruningtype, enum, discoverability, docs, source-dive]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# `pcg.set_self_pruning_settings` is undiscoverable from its docs: no enumerated `pruningType` values and no note on which params apply to which mode

The `pcg` wiki overlay (`docs/wiki-src/pcg.md:22`) documents
`pcg.set_self_pruning_settings` as exactly one line —
**"typed setter for the `UPCGSelfPruningSettings` node's parameters.
Source: `PCGSetSelfPruningSettings.cpp`."** — and enumerates **none** of:

1. the accepted `pruningType` enum values
   (`LargeToSmall | SmallToLarge | AllEqual | None | RemoveDuplicates`);
2. that `RadiusSimilarityFactor` is **only meaningful for the radius
   modes** (`LargeToSmall` / `SmallToLarge`) and is ignored by the others;
3. the `bRandomizedPruning` knob.

So when a caller is handed an intent in user vocabulary —
"set the pruning to a **radius-based** mode with a similarity factor
around 1.0 and randomized pruning on" — there is no `radius-based`
literal in the enum and no doc that maps the natural-language ask onto
the real value. The caller has to **read the engine header**
`PCGSelfPruning.h` to learn that the radius-based behaviour lives under
`LargeToSmall`/`SmallToLarge` (and that `RadiusSimilarityFactor` only
binds there) before it can pick the right `pruningType`.

This is the **docs/discoverability** angle and the same
**no-enumerated-values / overlay-omits-the-mapping** failure mode already
tracked for sibling namespaces:
[E-configure-slot-behavior-behaviortype-undiscoverable](E-configure-slot-behavior-behaviortype-undiscoverable.md)
(an `ai.configure_slot_behavior` `behaviorType` with no enumerated values
on its page, learned only by a wrong-guess retry + a C++ read) and
[E-state-tree-node-struct-type-undiscoverable](E-state-tree-node-struct-type-undiscoverable.md).
Here the un-enumerated arg is `pruningType` and the additional missing
fact is the param/mode applicability rule.

Distinct from the two PCG ergonomic/bug tickets nearby:
`E-pcg-add-node-echo-pin-labels` (node-creators omit pin labels) is about
*wiring*, not the self-pruning enum; `B-pcg-inspect-edge-direction-reversed`
is the inspect edge-direction bug; `F-pcg-filters-and-subgraphs` (DONE)
is the *feature* that shipped `set_self_pruning_settings` but its overlay
note never grew the value/mode documentation. The handler itself works
fine (the task's call succeeded first try once the value was known) — the
gap is purely that the right value isn't discoverable without the header.

## What it should do

Extend the `docs/wiki-src/pcg.md` overlay's `set_self_pruning_settings`
bullet to a short note that:
(a) enumerates the `pruningType` accepted values
(`LargeToSmall | SmallToLarge | AllEqual | None | RemoveDuplicates`),
mapping the common "radius-based" intent → `LargeToSmall`/`SmallToLarge`;
(b) states `RadiusSimilarityFactor` only applies to those two radius
modes (ignored by `AllEqual`/`None`/`RemoveDuplicates`);
(c) names the `bRandomizedPruning` boolean.
Downstream wiki-overlay edit (not applied here); no behaviour change —
the handler already writes the params correctly.

## Friction evidence (this task — namespace `pcg`, `PG_HillsideFoliageScatter` hillside-foliage build, 26 calls, outcome clean)

Foliage-scatter graph: SurfaceSampler → slope(NormalToDensity) →
spatial-noise(FBM) → self-pruning, wired Input→…→Output. The
`pcg.set_self_pruning_settings {SelfPruning_0, LargeToSmall, radius=1.0,
randomized=true}` call succeeded on the first try — but only after the
agent source-dived to map the user's words onto the enum.

Friction note (verbatim, partial): "the self-pruning enum has no literal
'radius-based' value (LargeToSmall|SmallToLarge|AllEqual|None|RemoveDuplicates)
— I had to read the engine `PCGSelfPruning.h` to learn `RadiusSimilarityFactor`
only applies to LargeToSmall/SmallToLarge and map 'radius-based' ->
LargeToSmall." The agent rated this "Mild … normal read-only discovery,
no python/plugin-source fallback needed" — i.e. no retry was burned, but
the *engine-header read* is exactly the source-dive this docs ticket
exists to remove. The clean outcome is despite, not because of, the docs.

## History
- `#1-initial-audit` `OPEN` reporter — Process/struggle audit of the `PG_HillsideFoliageScatter` hillside-foliage PCG build (namespace `pcg`, outcome clean, 26 calls, judge filed nothing — `filed_id` empty). Source-dive friction: the `pcg` overlay's `set_self_pruning_settings` bullet (`docs/wiki-src/pcg.md:22`) is one line ("typed setter for the UPCGSelfPruningSettings node's parameters") and enumerates no `pruningType` values, so an intent phrased "radius-based mode with similarity factor ~1.0, randomized on" had no literal match; the agent read engine `PCGSelfPruning.h` to learn the enum set (LargeToSmall|SmallToLarge|AllEqual|None|RemoveDuplicates), that `RadiusSimilarityFactor` only binds on the two radius modes, and to map "radius-based" → `LargeToSmall`. Call then succeeded first try. Same no-enumerated-values/overlay-omits-the-mapping failure mode as `E-configure-slot-behavior-behaviortype-undiscoverable` (ai) and `E-state-tree-node-struct-type-undiscoverable` (state_tree), applied to `pruningType`. E-/docs: works once the value is known; the gap is discoverability + a forced engine-header read. Deduped: ripgrep over OPEN+closed board for self.?prun / set_self_pruning / PruningType / radius-based found only `F-pcg-filters-and-subgraphs` (DONE feature that shipped the handler; its overlay note never documented values), `B-pcg-inspect-edge-direction-reversed` (inspect edge bug, unrelated), and `E-pcg-add-node-echo-pin-labels` (wiring pins, different friction) — none documents the self-pruning enum. Proposes the `docs/wiki-src/pcg.md` overlay enumerate `pruningType`, note the RadiusSimilarityFactor/radius-mode applicability, and name `bRandomizedPruning`.
