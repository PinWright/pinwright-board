---
id: F-static-mesh-slot-spatial-extent
title: "No verb reports per-triangle material id, per-slot spatial extent, or per-part triangle counts on a compiled mesh — a build's own claims about where geometry moved and how big a part is are unverifiable through the published surface"
status: OPEN
severity: Medium
category: feature
tags: [static-mesh, static-mesh-describe, asset-dump, sections, slotUsage, bounding-box, per-part, mesh-review, read-only, weapons]
encounters: 1
lastSeen: 2026-09-05T00:00:00Z
---

# The section map says how much geometry is on a slot; nothing says where it is

`F-static-mesh-section-material-map` shipped `sections[]` and `slotUsage[]`, which answered *how many
triangles* are on each material slot. The next question a mesh review asks — *where those triangles
are, and how big each authored part is* — has no published answer at all.

## What exists now

`static_mesh.describe` and the `static_mesh.json` sidecar carry, per LOD, `{index, materialIndex,
materialSlotName, firstIndex, numTriangles, minVertexIndex, maxVertexIndex, bEnableCollision,
bCastShadow}`, plus `slotUsage[]` with `lod0TriangleCount` / `lod0TriangleFraction`, plus a single
whole-mesh `bounds`.

## What is missing

1. **Per-slot spatial extent.** `slotUsage[]` should carry a bounding box (min/max, or
   origin+extent) computed over the vertices its sections reference. This is the natural follow-on to
   the shipped work: the row already exists, it already aggregates that slot's sections, and the
   vertex range (`minVertexIndex`/`maxVertexIndex`) is already emitted beside it — the box is the
   aggregate of data the builder has in hand.
2. **Per-triangle material id.** For any question finer than a slot ("does the barrel end at
   X = 33.8?"), there is no way to select a triangle subset by material. The section table gives
   contiguous index ranges, which is close, but a caller cannot get from a section to a coordinate
   without leaving the published surface.
3. **Per-part triangle counts.** A "part" here is an authored sub-object (a stock, a magazine, a
   grip), which after merge is not a slot and not a section — it is a spatially separated component.
   A connected-component / island breakdown with a triangle count and a box per island answers the
   review question directly and is a standard mesh operation.

## The consequence, measured this round

A build report (`build-03`) made three quantitative claims about the weapon meshes. **None of them
can be checked against any published verb:**

- **"Receiver forward reach 54.20 → 33.80"** — a per-slot extent claim. The only published extent is
  the whole-mesh `bounds`, which does not change when one slot's geometry is pulled back, so the
  claim is neither confirmable nor falsifiable.
- **"wall migration ±157 / ±238"** — a spatial displacement of geometry between regions. Nothing in
  the surface localises triangles at all.
- **"stock 386 → 420 triangles"** — a per-part count. The stock is not a material slot and not a
  section; `slotUsage[]` cannot isolate it, so the number is uncheckable even though the mesh is
  compiled and in hand.

The pattern is the one `F-static-mesh-section-material-map` was filed on, one level finer: a review
is asked to accept a build's own arithmetic because the readback cannot reproduce it. Read-only
verification is exactly the case this surface should serve.

## What is asked for

Land these on the **shared static-mesh builder** so `static_mesh.describe` and `static_mesh.json`
gain them together, per `E-dump-rpc-parity`:

- **Minimum, and the cheapest win:** `boundingBox` on each `slotUsage[]` row (LOD0), computed over
  the vertices of that slot's sections. One extra pass over data the builder already walks. This
  alone makes "did the receiver's forward reach change?" a diff of two numbers.
- **Next:** the same box per `sections[]` row, so a multi-section slot is separable.
- **Then:** an opt-in `islands[]` (connected components of LOD0) with `{triangleCount, boundingBox,
  materialSlots[]}` per island — the per-part answer. Opt-in because it is the only item here that
  costs a real traversal.
- **Not asked for:** a full per-triangle material array in the response. The per-triangle *question*
  should be answerable, but a 20k-row array is a response-spill problem; sections plus boxes plus
  islands answer it without one.

## Severity

**Medium**, feature. It is not a silent-wrong-data defect — nothing lies, the fields simply are not
there — but it is a hard stop for read-only mesh review: three specific claims went unverified this
round with no workaround short of `python.execute` and raw render data, which is the escape hatch
`F-static-mesh-section-material-map` was filed to remove.

## Related

- `F-static-mesh-section-material-map` (**DONE** — verified in use this round) — **the direct parent.**
  It made per-slot triangle *counts* readable; this asks for per-slot *position* and per-part
  granularity. Its own ask list already named "each slot's share of the mesh's triangles" as the
  derived thing reviews want; extent is the same argument one step on.
- `F-static-mesh-uv-channel-readout` (OPEN, Medium) — the third field wanted in the same builder. A
  fixer opening `StaticMeshDumpBuilder.cpp` should do all of these in one pass.
- `B-measure-verbs-aabb-and-actor-only` — the measurement family's existing limit to actor-level AABBs,
  which is why the whole-mesh `bounds` is the only extent available today.
- `B-pwmodel-per-part-uv-layout-overlaps-across-parts`, `B-pwmodel-modifier-output-takes-slot-zero`,
  `B-pwmodel-boolean-output-takes-slot-zero` — authoring-side defects whose effects are per-part and
  per-slot spatial, i.e. exactly what this readout would catch at review time.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Filed during a WEAPONS critic review round 3, as the measured follow-on to `F-static-mesh-section-material-map` (confirmed DONE the same round). With `sections[]` and `slotUsage[]` now published, per-slot triangle counts are readable, but no verb reports **per-triangle material id, per-slot spatial extent, or per-part triangle counts** on a compiled mesh — the only extent on the surface is a single whole-mesh `bounds`. Consequence measured this round: three quantitative claims in build-03 are unverifiable through the published surface — **"Receiver forward reach 54.20 → 33.80"** (a per-slot extent; the whole-mesh bounds does not move when one slot's geometry is pulled back, so the claim is neither confirmable nor falsifiable), **"wall migration ±157 / ±238"** (a spatial displacement; nothing localises triangles), and **"stock 386 → 420 triangles"** (a per-part count; the stock is neither a material slot nor a section, so `slotUsage[]` cannot isolate it). Ask, cheapest first, on the shared builder so `static_mesh.describe` and `static_mesh.json` gain it together per `E-dump-rpc-parity`: a `boundingBox` on each `slotUsage[]` row over that slot's sections' vertices — one extra pass over data already walked, and the row and its `minVertexIndex`/`maxVertexIndex` are already emitted; then the same box per `sections[]` row; then an opt-in `islands[]` (LOD0 connected components) with `{triangleCount, boundingBox, materialSlots[]}` for the per-part answer. Explicitly **not** asking for a per-triangle material array in the response — a 20k-row field is a spill problem, and sections + boxes + islands answer the per-triangle question without one. Severity **Medium**, feature: nothing lies, the fields are absent, but read-only mesh review has no workaround short of `python.execute` over raw render data — the escape hatch the parent ticket was filed to remove. Should land with `F-static-mesh-uv-channel-readout` in the same builder pass.
