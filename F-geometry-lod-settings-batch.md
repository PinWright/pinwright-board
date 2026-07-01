---
id: F-geometry-lod-settings-batch
title: "No batch form for geometry.set_lod_settings — N-1 calls to configure an N-LOD reduction ladder (asymmetric with array-based set_lod_screen_sizes)"
status: OPEN
severity: Low
category: feature
tags: [geometry, lod, set_lod_settings, generate_lods, batch, static-mesh]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# Per-LOD reduction config has no batch form; the canonical "LOD'd prop" intent costs N+1 calls

`geometry.set_lod_settings` configures exactly **one** LOD level per call —
`lodIndex` is a scalar, and the reduction knobs (`trianglePercent`,
`recomputeNormals`, `recomputeTangents`) apply only to that single index. To
build the canonical "generate an N-LOD asset where each LOD has a distinct
reduction percentage" result, an agent must issue:

- 1× `geometry.generate_lods` (lodCount=N), then
- (N-1)× `geometry.set_lod_settings` (one per non-zero LOD), then
- 1× `geometry.set_lod_screen_sizes` (this one already takes the **whole**
  `screenSizes` array at once).

That last point is the tell: the sibling method `set_lod_screen_sizes` accepts
`screenSizes` (`array`, required) and configures **all** LOD transitions in a
single call, but the reduction-settings method has no parallel array form. The
two halves of "set up the LOD ladder" are asymmetric — screen sizes are
one-shot, per-LOD reduction is one-call-per-level.

## Why this is friction (not just normal granularity)

"Build a reusable prop and turn it into a proper LOD'd Static Mesh" is a
recurring, canonical authoring intent — the user expressed the reduction tuning
as a *single* conceptual step ("make LOD1 keep ~50%, LOD2 ~25%, LOD3 ~10%,
recomputing normals on the aggressive levels"), but it decomposed into 3
separate RPCs that differ only in their per-row values. The cost scales with
LOD count and is paid every time a LOD'd mesh is authored. It also creates a
discoverability inconsistency: an agent that just learned `set_lod_screen_sizes`
takes an array will reasonably expect `set_lod_settings` to as well, then has to
fall back to looping when it does not.

This is a clean-outcome PROCESS finding: every call in the task succeeded first
try with no retries (see evidence below). The friction is the call *count* and
the array-vs-scalar asymmetry, not any failure.

## What it should do

Add a batch / array form so the whole reduction ladder is one call. Either:
1. **Plural batch handler (preferred, matches `F-batch-pin-defaults` precedent):**
   `geometry.set_lod_settings` accepting `lods: [{lodIndex, trianglePercent,
   recomputeNormals?, recomputeTangents?}, ...]`, applied under one transaction
   with a per-item `results` array (mirrors how `blueprint.graph.set_pin_default_values`
   was added next to the singular setter). Keep the scalar form for backward compat.
2. **Fold reduction into `generate_lods`:** let `generate_lods` accept an
   optional `reductionByLod` array so "generate N LODs at these percentages with
   these recompute flags" is a single call — symmetric with how
   `set_lod_screen_sizes` already takes the full array.

Either collapses the canonical N-LOD setup from N+1 calls to ~2 (generate +
screen-sizes) or even 1.

## Friction evidence (this task — geometry.generate_lods "BoulderProp" build, 10 calls, outcome clean)

Story: build a 4-LOD "BoulderProp" StaticMesh with per-LOD reductions
(50/25/10%, recompute normals on LOD2/3) and screen sizes [1,0.5,0.25,0.1].
Call log shows the reduction config split across **three** consecutive
single-index calls:

- `geometry.set_lod_settings {lodIndex=1 trianglePercent=50 recomputeNormals=false}` → ok
- `geometry.set_lod_settings {lodIndex=2 trianglePercent=25 recomputeNormals=true}` → ok
- `geometry.set_lod_settings {lodIndex=3 trianglePercent=10 recomputeNormals=true}` → ok

while the very next step set all four screen-size transitions in **one** call:

- `geometry.set_lod_screen_sizes {screenSizes=[1,0.5,0.25,0.1]}` → ok

The agent's friction note was *"none — wiki pages for every geometry.* method
were on disk and gave exact param names; all RPCs succeeded first try with no
retries, workarounds, or python.execute fallback."* So nothing errored — this is
pure call-count/asymmetry overhead on an otherwise clean run, surfaced by the
struggle audit rather than the per-finding judge (which filed nothing,
`filed_id` empty).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.generate_lods` "BoulderProp" task (10 calls, outcome clean, judge filed nothing). PROCESS finding: `geometry.set_lod_settings` (wiki-generated/geometry.set_lod_settings.md — scalar `lodIndex` + per-level `trianglePercent`/`recomputeNormals`/`recomputeTangents`) configures one LOD per call, so an N-LOD reduction ladder costs N-1 such calls (here 3: lodIndex 1/2/3 at 50/25/10%). Its sibling `geometry.set_lod_screen_sizes` (wiki-generated/geometry.set_lod_screen_sizes.md) already takes the whole `screenSizes` array in one call, making the LOD-setup surface array-vs-scalar asymmetric. No retries/errors/fallbacks — pure call-count overhead. Proposed: add a batch `lods:[{lodIndex,...}]` form (mirrors implemented `F-batch-pin-defaults` plural handler) or fold a `reductionByLod` array into `geometry.generate_lods`. Dedup: no existing LOD ticket on the board; geometry E-tickets cover unrelated surfaces (auto_uv/unwrap_uv duplication, deformer count echoes, create name-vs-actorName, mesh-info bbox).
