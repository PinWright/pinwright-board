---
id: F-landscape-paint-region-shapes
title: "landscape.create_procedural_terrain's region is an axis-aligned rectangle in heightmap pixels and nothing else — no brush, polyline, slope mask or height mask — so 'paint the layer to follow the landform' is not expressible"
status: OPEN
severity: Medium
category: feature
tags: [landscape, create_procedural_terrain, layer-paint, region, weightmap, brush, mask, slope, shape-expressiveness]
blockedBy: [B-paint-layer-destroys-other-layer-weights]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The only shape a layer paint can take is a box

`landscape.create_procedural_terrain` (`LandscapeHandler.cpp:1981`) is the
plugin's layer-paint verb — its own summary calls it "a misnomer-named alias for
layer paint". Where it paints is decided by exactly one parameter, `:1987`,
reproduced verbatim in `Saved/PinWright/wiki/landscape.create_procedural_terrain.md:15`:

> `region` (`object`, optional): Heightmap-pixel region {minX, minY, maxX, maxY}
> to paint. Each coordinate independently defaults to the corresponding
> full-landscape extent and is clamped into it; a region that is empty after
> clamping is rejected with INVALID_ARGUMENT rather than painting nothing and
> reporting success.

Four integers. The wiki restates the model at `:37`: "`region` coordinates are
**landscape heightmap pixels**, not world units." The complete parameter list at
`:1983-1989` is `landscapePath`, `landscapeName`, `layerName`, `strength`,
`region`, `skipFlush`, `verify` — so `strength` is a single scalar applied
uniformly across that box, and there is **no brush radius, no falloff, no
polyline or spline, no polygon, no mask texture, no slope predicate, no height
predicate, and no per-texel weight array**.

The shapes terrain painting is actually made of are therefore not expressible:

- "sand below Z = 200, rock above" — a **height** mask.
- "cliff material wherever the slope exceeds 40 degrees" — a **slope** mask, the
  single most common landscape layer rule there is.
- "a 6 m dirt path along this polyline" — a **stroke**, which the height-writing
  sibling `landscape.sculpt` already takes (`path`, `:836`) and this verb does not.
- "grass everywhere except the lake footprint" — a **carve-out**.

The nearest expressible thing is a box, which reads as a box.

## Two costs a reviewer will raise — both are real

**(a) The region model is shared, and by more verbs than its own docs admit.**
`region` resolves through `LandscapeHeightStats::ResolveHeightRegion`
(declared `LandscapeHeightStats.h:55`, defined `LandscapeHeightStats.cpp:17`),
which takes four `TOptional<int32>` plus the landscape's full extent and returns
a clamped `FResolvedHeightRegion` of four ints. It has **four** call sites, not
the two its own header comment claims and not the three a quick read suggests:

| line | verb |
|---|---|
| `LandscapeHandler.cpp:1735` | `landscape.edit` |
| `LandscapeHandler.cpp:1917` | `landscape.get_heights` |
| `LandscapeHandler.cpp:2173` | `landscape.create_procedural_terrain` |
| `LandscapeHandler.cpp:2519` | `landscape.audit_shape` (registered `:2414`) |

The header's doc comment at `LandscapeHeightStats.h:44-46` says it is "shared by
the read verb (`landscape.get_heights`) and the write verb (`landscape.edit`)" —
that comment is already stale by two verbs, which is itself worth fixing while
anyone is in here. So a richer region model is a **four-verb** change, and the
struct it returns (four ints + `bValid`) cannot carry a mask or a predicate
without changing shape. That argues for an **additive** design rather than a
replacement: keep `region` exactly as it is, and add a separate optional
`mask` / `shape` input that composes with it (rectangle stays the bounding box;
the mask decides per-texel weight inside it). Then `get_heights`, `edit` and
`audit_shape` are untouched and only the paint verb grows.

**(b) "Use PCG instead" is not a substitute, and I checked rather than assumed.**
`F-pcg-filters-and-subgraphs` (DONE, Medium) is the obvious counter — it shipped
`pcg.add_slope_filter` and `pcg.add_noise_filter`, and "slope filter" is the exact
phrase this ticket wants. I read it in full. Those helpers spawn
`UPCGNormalToDensitySettings` and `UPCGSpatialNoiseSettings` /
`UPCGAttributeNoiseSettings`: they filter **PCG points** by density derived from
surface normals, inside a PCG graph, to decide where instances get spawned. The
file contains no reference to landscapes, weightmaps, weight layers,
`LandscapeLayerBlend`, `SetAlphaData` or the `landscape.*` namespace at all — the
only reason it looks relevant is the word "slope".

The two operate on different data and produce different output. PCG decides
**where meshes go**. A landscape layer paint writes **weightmap texels** on
`ULandscapeComponent`, which the landscape material's `LandscapeLayerBlend`
consumes to decide which texture the ground *is*. Nothing on the shipped PCG
surface writes a landscape weightmap, and no arrangement of point filters
produces one. PCG can scatter rocks on a slope; it cannot make the slope read as
rock. So the counterargument does not survive contact — but it is close enough
to the surface that this ticket should carry the refutation rather than wait to
be asked.

## Workaround

Any shape decomposes exactly into axis-aligned rows: scanline-rasterise the mask
and issue one `create_procedural_terrain` call per run of texels. It is exact,
not an approximation. It costs up to one call per heightmap row (505 on the
default landscape), each with its own edit-layer settle unless `verify: false` /
`skipFlush: true` is used to batch. That is the rubric's "many extra calls", and
it is why this is Medium rather than High.

The workaround survives the destructive-paint defect below, incidentally: that
ticket reports *other* layers being zeroed, so repeated rectangles of the **same**
layer compose correctly. Painting a second layer is what does not.

## Deferral

`blockedBy: [B-paint-layer-destroys-other-layer-weights]`. That ticket (Critical,
OPEN, filed this session) reports that one paint at strength 1.0 over a sub-region
leaves every *other* target layer at zero weight across the **whole** landscape,
measured twice on a running editor — and that the verb's own `verify` block reads
only the layer it just painted over only the rectangle it just painted, so
`texelsWithWeight == paintedTexels` reads green over the destruction.

The interaction is plain: the entire point of expressive region shapes is
**multi-layer** terrain — grass here, rock on the cliffs, sand at the waterline.
While the second paint erases the first, a caller cannot get to two layers at all,
so a better shape for one of them buys very little. Fix the destruction first.

Stated honestly so a triager can overrule it: this is a gate on **value**, not on
feasibility. A richer region model is implementable today and would help
single-layer work (a slope-masked path, a height-masked shoreline) even with the
destruction unfixed. If the picker would rather have the shape work in parallel,
drop the `blockedBy` — the argument for ordering, not the argument for the
feature, is what it encodes.

## What it should do

Add an optional per-texel weight source that composes with the existing
rectangle, leaving `region` and `ResolveHeightRegion` untouched:

1. **`slopeRange` / `heightRange`** `{min, max}` with an optional falloff width —
   the two masks that cover most real landscape material rules, both computable
   from the heightfield the verb can already read (`get_heights` uses
   `FLandscapeEditDataInterface::GetHeightData` over the same region).
2. **`path` + `brushRadius` + `brushFalloff`** — the stroke model
   `landscape.sculpt` already implements at `:836-839`, applied to weight instead
   of height. Same distance-field rasterisation, same parameters, so the two verbs
   would finally describe shape the same way.
3. **`maskTexture`** — a texture/render-target asset path sampled across the
   region, which is the general case and subsumes anything computed offline.

Whatever lands should echo the texels actually written (the verb already reports
`paintedTexels` / `texelsWithWeight` / `texelsAtRequestedWeight`), so a mask that
selected nothing is visible rather than a clean-looking no-op.

severity rationale: impact — a missing capability on a shipped verb, but not a
hard blocker: scanline decomposition into axis-aligned rows expresses any shape
exactly, at up to ~505 calls on the default landscape. That is the rubric's
Medium proper, "doable, but only via a documented workaround ... or many extra
calls". Reach: layer painting is not an almost-every-session method, so I
**decline the reach bump-up** to High; nor is it a rare edge path — it is the
only layer-paint verb in the namespace and the only way to make terrain read as
anything but one material — so I **decline the bump-down** to Low. -> **Medium**.

## Same shape as

- `B-create-procedural-terrain-paints-nothing` (DONE, High) — same verb, same
  parameter, **different axis**: whether the rectangle is *valid and honest*, not
  whether a rectangle is the only sayable shape. Its `#2` made the region correct
  — replaced the `-1` sentinel with `TOptional` + the shared `ResolveHeightRegion`,
  clamped each coordinate independently into the extent, reported any clamp that
  changed the request in `warnings[]`, and rejected an empty or inverted region
  with `INVALID_ARGUMENT` naming the actual extent instead of painting nothing and
  returning success (`#3` verified live: `{minX:100,maxX:1}` → `INVALID_ARGUMENT`).
  That work is exactly right and this ticket depends on it. I read the file in
  full: **nothing in it asks whether a rectangle is enough.** Every region
  discussion is about the box's validity — emptiness, inversion, negative
  coordinates, bounds — never its expressiveness. No mention of circles, polygons,
  masks, splines, falloff or feathering anywhere.
- `B-paint-layer-destroys-other-layer-weights` (OPEN, Critical, filed this
  session) — same verb, **different axis**: what a paint does to layers it was not
  asked to touch, and to texels outside the region it was given. See the Deferral
  section above for the interaction; it is the ticket this one is gated on.

## History
- `#1-rectangle-is-the-only-shape` `OPEN` reporter — Filed from source, every citation re-derived at HEAD in this checkout; not RPC-replayed in this session. `landscape.create_procedural_terrain` (`LandscapeHandler.cpp:1981`) takes `region` as four heightmap-pixel integers and nothing else (`:1987`, quoted verbatim in the body from `landscape.create_procedural_terrain.md:15`); the full param list `:1983-1989` has no brush, path, polygon, mask, slope or height input, and `strength` is one uniform scalar. Cost (a): the region model is shared via `LandscapeHeightStats::ResolveHeightRegion` (`LandscapeHeightStats.h:55`, `.cpp:17`) by **four** verbs, not the two its own header comment at `.h:44-46` names — `landscape.edit` `:1735`, `landscape.get_heights` `:1917`, this verb `:2173`, and `landscape.audit_shape` `:2519` — so the fix should be additive (a mask that composes with the rectangle) rather than a change to the shared resolver; the stale header comment is worth correcting in passing. Cost (b): read `F-pcg-filters-and-subgraphs` (DONE) in full to test the "use PCG instead" counterargument — its `pcg.add_slope_filter` / `add_noise_filter` spawn `UPCGNormalToDensitySettings` / `UPCGSpatialNoiseSettings`, which filter PCG points by normal-derived density to place instances; the file mentions no landscape, weightmap, weight layer, `LandscapeLayerBlend` or `SetAlphaData` anywhere. PCG decides where meshes go; a layer paint writes the weightmap texels that decide what the ground *is*. Not a substitute. Workaround: scanline-decompose any shape into axis-aligned runs, exact but up to ~505 calls on the default landscape — hence Medium. Deferred `blockedBy: [B-paint-layer-destroys-other-layer-weights]` (Critical, OPEN): expressive shapes exist to serve multi-layer terrain, and while a second paint zeroes every other layer landscape-wide a caller cannot reach two layers at all — a value gate, not a feasibility gate, and flagged as droppable if a triager disagrees. Cross-linked to `B-create-procedural-terrain-paints-nothing` (DONE, High), whose `#2` made this same rectangle correct and honest and which — confirmed by reading the whole file — never once asks whether a rectangle is the only expressible shape.
