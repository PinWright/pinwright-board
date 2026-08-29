---
id: F-landscape-noise-and-erosion
title: "landscape.sculpt has only Raise/Lower/Flatten/Smooth — no noise and no erosion anywhere in the landscape namespace, so terrain has no sub-metre roughness; the plugin already ships in-process Perlin noise for dynamic meshes and FBM noise for textures, just not for the one surface that IS a heightfield"
status: OPEN
severity: Medium
category: feature
tags: [landscape, sculpt, noise, erosion, roughness, heightfield, coverage-gap, brush-repertoire]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The brush repertoire has four modes and every one of them smooths

`landscape.sculpt` (`LandscapeHandler.cpp:830`) declares its tool set at `:837`,
verbatim (mirrored in `Saved/PinWright/wiki/landscape.sculpt.md:16`):

> `toolMode` (`string`, optional): Sculpt operation: 'Raise', 'Lower', 'Flatten'
> or 'Smooth'. Defaults to 'Raise'. An unrecognised value is refused with
> LANDSCAPE_INVALID_TOOL_MODE, never silently ignored.

That is the complete repertoire. The shaping parameters around it —
`brushRadius` `:838`, `brushFalloff` `:839`, `falloffProfile` `:840`
(linear / smooth / spherical / tip), `strength` `:841`, `heightDelta` `:842`,
`targetHeight` `:843`, `smoothRadiusVerts` `:845` — are all about *where* one
smooth dome of displacement lands, never about texture within it. Raise and
Lower add a falloff dome. Flatten removes variation. Smooth removes it faster.
**No mode adds detail.**

Nor does anything else in the namespace. It registers exactly eight verbs —
`create` `:387`, `sculpt` `:830`, `set_material` `:1410`, `create_grass_type`
`:1502`, `edit` `:1626`, `get_heights` `:1840`, `create_procedural_terrain`
`:1981`, `audit_shape` `:2414` — and `landscape.edit`'s operations (`:1630`) are
`'set'`, `'raise'`, `'lower'`, `'flatten'`. A case-insensitive grep for
`erosion|erode|perlin` across the whole of `Handlers/Environment/` returns
**zero** hits.

The observable consequence on this project's test level: terrain is smooth below
roughly 10 m. Macro landform is achievable — a stroke-swept Raise makes a ridge,
Flatten makes a pad — but everything under a brush radius is a mathematically
smooth falloff surface, so the ground reads as a modelling clay maquette rather
than terrain. (Recorded as motivation from content work in this checkout, not as
a measured plugin metric.)

## The plugin already does this, in-process, twice — just not for the heightfield

This is a coverage gap with two shipped precedents inside the same plugin:

| verb | source | what it does |
|---|---|---|
| `geometry.noise_deform` | `PinWrightGeometry/.../MeshOpsHandler.cpp:1380` | "Apply Perlin noise deformation to a dynamic mesh" — displaces vertices in-process |
| `texture.create_noise_texture` | `PinWright/.../TextureHandler.cpp:2731` | "Create a procedural FBM noise texture (Perlin value noise or Worley/Voronoi cellular)" — generates in-process |

So the plugin can already perturb a mesh's vertices with Perlin noise and can
already synthesise FBM/Worley fields. The **landscape** — the one asset in the
engine that literally *is* a heightfield, and the one where noise is not a
stylistic flourish but the difference between terrain and a ramp — has neither.
Both precedents also establish the shape of the answer: the noise is computed
where the geometry lives, and nothing but parameters crosses the wire.

## Two framings that pull in opposite directions, and how they resolve

**(a) A pure computational generator is the cheap thing to build.**
`F-scatter-layout-verb` (DONE, Medium) is the board's precedent and makes the
argument under a heading literally titled "The ask: a pure function, no side
effects":

> "Three properties make this cheap and worth having: 1. **It is a pure
> function.** No world access, no actors, no assets, no transaction. It can be
> unit tested in `Tests/Spatial/` with no fixture at all... 3. **It is testable
> without an editor**, which matters given how much of this domain currently is
> not."

Its `#2` shipped exactly that — `spatial.scatter_layout`, a new file, no world,
no transaction, five tests needing no editor fixture. By that precedent an
erosion/noise **heightfield generator** looks like a far better fit than a new
brush mode: hydraulic erosion is a lot of arithmetic and no engine state, and it
unit-tests against a known input buffer.

**(b) But a generator's output has to get back in, and that path is blocked.**
`spatial.scatter_layout` composes cleanly because its output is a few thousand
transforms that the placement verbs already consume. A heightfield generator's
output is 255,025 uint16 samples for the default landscape, and the only way to
install it is `landscape.edit operation='set'` with `heightData` as an inline
array — the gap filed as `F-landscape-edit-height-data-source`.

**Erosion is worse than noise here, because it also needs the terrain as input.**
Hydraulic and thermal erosion are functions of the *existing* heightfield, so a
pure generator would have to read the surface out first. That read is not blocked
by a cap — `landscape.get_heights`' `maxSamples` treats `<= 0` as "return all"
(`LandscapeHandler.cpp:1887`, applied `LandscapeHeightStats.cpp:86`) — but it is
blocked by the same volume: 255,025 numbers out, compute, 255,025 numbers back.
Pure noise needs no read and is blocked at one end; erosion is blocked at both.

## Decision: build it in-process. No defer, no `blockedBy`.

I am **not** setting `blockedBy: [F-landscape-edit-height-data-source]`, and the
reason is (b) rather than in spite of it. A read-modify-write over a quarter of a
million samples should never cross an RPC boundary in either direction — and it
does not have to, because the handler already holds an
`FLandscapeEditDataInterface` over exactly that data. Both existing precedents
(`geometry.noise_deform`, `texture.create_noise_texture`) compute in-process for
precisely this reason. An in-process implementation is blocked by nothing, ships
independently, and moves zero samples over the wire, so gating it behind the
height-source ticket would defer work that has no dependency on it. Setting
`blockedBy` here would be a defer that names a blocker which is not actually
blocking — and the README is explicit that a defer must name a *real* gate.

`F-landscape-edit-height-data-source` remains worth fixing on its own merits
(installing a DEM tile, an offline-authored heightmap, a render target) — it is
just not this ticket's gate. Cross-linked, not depended on.

The purity argument from (a) survives intact, one layer down: the noise/erosion
**kernel** should still be a pure function over a `TArray<uint16>` in its own
translation unit, unit-tested against a known buffer with no editor fixture,
exactly as `F-scatter-layout-verb` argued. Only the thin verb around it touches
the landscape. That gets the cheap tests and the working install path.

## What it should do

The two capabilities want different shapes and should not be forced into one:

1. **Noise as a `landscape.sculpt` `toolMode`.** Noise is a local, brush-shaped
   perturbation, so it fits the existing stroke model exactly: the same
   distance-field footprint, the same `brushRadius` / `brushFalloff` /
   `falloffProfile`, the same honest `modifiedVertices` post-write readback, and
   the same `LANDSCAPE_SCULPT_NO_CHANGE` refusal. Add `toolMode: "Noise"` plus
   `noiseScale` (feature size in world cm), `noiseAmplitude` (cm), `octaves`,
   `lacunarity`, `persistence` and `seed`. Fixing the "smooth below 10 m" symptom
   is one call: sweep a low-amplitude, small-scale FBM over the region.
   Note `B-noise-texture-noisetype-ignored` and `B-noise-texture-octaves-zero-nan`
   are open against the *texture* noise verb — whatever kernel this reuses should
   not inherit those defects.
2. **Erosion as its own region-based verb, not a brush mode.** Hydraulic and
   thermal erosion are iterative simulations over a region with an iteration
   count, a rainfall/talus-angle model and a sediment field. Sweeping that along
   a stroke would be wrong — it has no brush centre and no falloff. Propose
   `landscape.erode` taking `region` (the existing `ResolveHeightRegion` shape),
   `mode` (`hydraulic` | `thermal`), `iterations`, and the model constants, with
   the same settle/readback honesty the other height writers now have.

severity rationale: impact — a missing verb, which the rubric puts in the
"High or Medium: hard blocker with no workaround" band. I place it at **Medium**,
not High, because the blocked thing is *detail*, not *terrain*: the four existing
modes do produce usable macro landform, and `python.execute`
(`PythonExecuteHandler.cpp:116`) reaches `FLandscapeEditDataInterface` directly
for anyone willing to write engine Python — the same escape hatch
`F-landscape-height-readback` `#2` used to correct its own "Workaround: none".
That is Medium's "doable, but only via ... a source dive". Reach: terrain
sculpting is not an almost-every-session method, so I **decline the reach
bump-up** to High. I also **decline the rare-edge-path bump-down** to Low: within
terrain authoring this is not an edge case but the central missing capability —
no other verb in the namespace adds detail at any scale — and Low is reserved for
docs, naming and discoverability friction, which this is not. -> **Medium**.

## Explicitly NOT the same as

- `B-landscape-sculpt-stale-bounds` (IN-REVIEW, Medium) — **unrelated; do not
  merge these.** Same verb name in the title, nothing else in common. That ticket
  is a deferred-edit-layer synchronisation bug: `SetHeightData` + `Flush` only
  *schedule* the layer regeneration, so `CachedLocalBox` / `actor.get_bounding_box`
  still report Z = 0 in the very next RPC after a real raise, until
  `ALandscape::ForceUpdateLayersContent()` drives it to completion (its `#3` fix).
  It is entirely about **when the bounds refresh after a write**, and says nothing
  about which brushes exist — its repro uses `toolMode: "Raise"` and would behave
  identically under any mode. This ticket is about the **brush repertoire**: which
  height transformations the verb can express at all. A triager merging them would
  produce a ticket where fixing either half leaves the other untouched.

## Same shape as

- `F-scatter-layout-verb` (DONE, Medium) — the pure-generator precedent quoted
  above, and the reason the noise/erosion **kernel** should be a fixture-free unit
  under test even though the **verb** must run in-process.
- `F-landscape-edit-height-data-source` (OPEN, Medium, filed this session) — the
  reason an offline-generator design would not compose. Cross-linked as context;
  deliberately **not** set as `blockedBy`, per the Decision section.

## History
- `#1-no-noise-no-erosion` `OPEN` reporter — Filed from source, every citation re-derived at HEAD in this checkout; not RPC-replayed in this session. `landscape.sculpt` (`LandscapeHandler.cpp:830`) offers exactly Raise / Lower / Flatten / Smooth (`:837`), and every other shaping param (`brushRadius` `:838`, `brushFalloff` `:839`, `falloffProfile` `:840` linear|smooth|spherical|tip, `strength` `:841`, `smoothRadiusVerts` `:845`) positions one smooth falloff dome — no mode adds detail. The namespace's other seven verbs (`:387`, `:1410`, `:1502`, `:1626`, `:1840`, `:1981`, `:2414`) add none either, `landscape.edit`'s operations are set/raise/lower/flatten (`:1630`), and grep for `erosion|erode|perlin` across `Handlers/Environment/` returns zero. Motivation from content work in this checkout: terrain reads smooth below roughly 10 m. Strongest supporting fact, found while verifying: the plugin ALREADY ships in-process noise twice — `geometry.noise_deform` ("Apply Perlin noise deformation to a dynamic mesh", `PinWrightGeometry/.../MeshOpsHandler.cpp:1380`) and `texture.create_noise_texture` ("procedural FBM noise ... Perlin value noise or Worley/Voronoi cellular", `TextureHandler.cpp:2731`) — so the one surface that is literally a heightfield is the one that has neither. Framing (a): `F-scatter-layout-verb` (DONE) is the pure-generator precedent, quoted verbatim in the body ("It is a pure function ... unit tested ... with no fixture at all ... testable without an editor"). Framing (b): a generator's output is 255,025 samples for the default landscape and the only install path is `landscape.edit`'s inline `heightData` array (`F-landscape-edit-height-data-source`); erosion is worse because it needs the terrain as INPUT too — not capped (`maxSamples <= 0` means return-all, `:1887` / `LandscapeHeightStats.cpp:86`) but the same volume in the other direction. **Resolved by skipping the defer: no `blockedBy` set.** A read-modify-write over a quarter-million samples should not cross an RPC boundary at all, the handler already holds an `FLandscapeEditDataInterface` over that data, and both existing noise precedents compute in-process — so an in-process implementation has no dependency on the height-source ticket, and gating it there would name a blocker that is not blocking. The purity argument survives one layer down: the kernel stays a pure function over a `TArray<uint16>`, fixture-free under test; only the thin verb touches the landscape. Proposed shape: noise as a `landscape.sculpt` `toolMode: "Noise"` (brush-shaped, reuses the stroke footprint and the `modifiedVertices` readback), erosion as a separate region-based `landscape.erode` (iterative, no brush centre, no falloff). Confirmed by reading it in full that `B-landscape-sculpt-stale-bounds` (IN-REVIEW, Medium) is **unrelated** and must not be merged with this: it is a deferred-edit-layer bounds/collision refresh bug (`ForceUpdateLayersContent`), about when bounds refresh after a write, not about which brushes exist — its repro uses Raise and would behave identically under any mode. Dedup: no ticket on the board asks for landscape noise or erosion; the `noise` hits are `B-noise-texture-noisetype-ignored` / `B-noise-texture-octaves-zero-nan` (texture verb defects, cited here only as kernel-reuse hazards), `geometry.noise_deform`, `material.authoring.add_noise` and `pcg.add_noise_filter` — none of which touches a landscape heightmap.
