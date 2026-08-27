---
id: B-noise-texture-noisetype-ignored
title: "`texture.create_noise_texture` parses `noiseType` into a dead local and never branches on it — every value produces the same Perlin FBM, and an unknown algorithm name is accepted with success"
status: OPEN
severity: High
category: bug
tags: [texture, create_noise_texture, noise, parameter-ignored, silent-noop, false-success, unvalidated-input, voronoi]
encounters: 1
lastSeen: 2026-08-27T18:47:15+05:00
---

# `texture.create_noise_texture` declares `noiseType`, parses it, and never reads it again — all values produce identical Perlin FBM and a nonsense value is accepted

`texture.create_noise_texture` declares a `noiseType` parameter documented as
"Noise algorithm (e.g. Perlin)". The handler parses it into a local and then
never branches on it. Every value — `Perlin`, `Voronoi`, or a string that names
no algorithm at all — produces byte-identical Perlin FBM, and the call returns
`success` in every case.

Two failures compound:

1. **The parameter is inert.** A caller who asks for cellular noise gets Perlin,
   with nothing in the response saying so.
2. **An unrecognised value is not refused.** `noiseType: "ZZZNotARealNoise"`
   creates the asset with no error and no warning. The caller's typo, or their
   belief that an algorithm exists, is silently converted into a success.

## Root cause (guilty source line)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureHandler.cpp:244`:

```cpp
    FString NoiseType = GetStringFieldTextAuth(Params, TEXT("noiseType"), TEXT("Perlin"));
```

`NoiseType` occurs exactly twice in the entire file: this parse, and the
parameter declaration at `TextureHandler.cpp:2517`:

```cpp
    RPC_PARAM_DEF("noiseType", "string", "Noise algorithm (e.g. Perlin)", "Perlin")
```

It is never compared, switched on, or passed anywhere. The pixel loop at
`TextureHandler.cpp:275-311` calls `FBMNoise(...)` unconditionally on both the
seamless and non-seamless branches. This is a write-once dead local, provable
statically without running anything.

## Verbatim repro

Three calls identical but for `noiseType`, then `texture.get_pixel_stats` on the
first two:

```
texture.create_noise_texture {name:"A", width:512, height:512, scale:6, octaves:1,
                              seed:7, seamless:true, noiseType:"Voronoi"}
texture.create_noise_texture {name:"B", width:512, height:512, scale:6, octaves:1,
                              seed:7, seamless:true, noiseType:"Perlin"}
texture.create_noise_texture {name:"C", width:512, height:512, scale:6, octaves:1,
                              seed:7, seamless:true, noiseType:"ZZZNotARealNoise"}
```

A and B return **byte-identical content**: `hash: "5c4b7e25"`,
`mean.r 116.64346313476562`, `min 44`, `max 240` on both. C is created with no
error and no warning.

## Reach: there is no cellular generator anywhere in the plugin

The parameter is not merely unimplemented for one value — the capability it
advertises does not exist on any surface:

- `material.graph.search_expression_types {query:"Voronoi"}` returns **0 results**.
- The only cellular node available is `MaterialExpressionVectorNoise`, which is a
  per-pixel material node, not a texture lookup.

So an author who needs a tiling cellular texture — caustics, cracked stone,
scales, cobble, dragon skin — has no route through the plugin, and the one
parameter that names the capability answers `success`.

## The plugin already does this correctly one namespace over

`Plugins/PinWright/Source/PinWright/Private/Handlers/PCG/PCGAddNoiseFilter.cpp:135`
parses its own `noiseType`, branches on it, and rejects an unknown value with
`"Unknown spatial noiseType: %s"`. The rejection pattern this ticket asks for is
already written and shipping in the same codebase.

## What it should do

Either implement the algorithms the parameter advertises (at minimum a
Worley/Voronoi F1 distance field, which is what `Voronoi` would mean here), or
refuse an unrecognised `noiseType` with `INVALID_ARGUMENT` naming the supported
set — following `PCGAddNoiseFilter.cpp:135`. Both is better: refusing unknown
values is a one-line fix that stops the silent lie today, and the generator can
land later.

## Workaround

Build the cellular pattern out of Perlin. Generate FBM, then `texture.adjust_curves`
with a V-shaped master curve centred on the measured mean, which turns the
mid-value iso-contour into bright thin closed cells. Exact recipe used for
`/Game/Atlantis/Textures/T_Caustic_Cells` on the Atlantis build: a
`scale:7, octaves:2, persistence:0.4` base, then master curve
`input [0, 0.33, 0.43, 0.49, 0.55, 0.65, 1]` to
`output [0, 0, 0.08, 1, 0.08, 0, 0]`. The result is a genuine caustic web. This
is a workaround, not a substitute — it has no cell-ID or F2-F1 output.

## Distinct from related tickets

- `B-texture-create-placeholder-fake-success` (IN-REVIEW, High) is the same
  *shape* — a success-returning no-op — but scoped to
  `create_texture_array`/`create_cube_texture`/`create_volume_texture`, which
  create no asset at all. `create_noise_texture` genuinely creates a real
  texture; only the algorithm choice is fake.
- `E-texture-action-handler-param-docs` (IN-REVIEW) is about `texture.*` action
  handlers hardcoding `RPC_NO_PARAMS` so the wiki says "Parameters: none".
  `create_noise_texture` **does** declare `noiseType` properly via
  `RPC_PARAM_DEF` at `:2517` — the defect here is that the declared, documented
  parameter is never read, not that it is undocumented.
- `B-texture-save-no-disk-write` (IN-REVIEW, Critical) names
  `create_noise_texture` at `TextureHandler.cpp:305`, but its fault is
  `McpSafeAssetSave` marking dirty without writing bytes. Orthogonal: our
  textures existed on disk and read back correctly — the *pixels* are wrong, not
  the persistence.
- `B-noise-texture-seamless-lattice` is the sibling pixel defect in the same
  verb (the `seamless:true` tiling path), found in the same session. Different
  guilty lines (`:284-292` vs `:244`); fix them together while the file is open.

severity rationale: impact=silent false-success on a normal path (a documented parameter is inert and an unknown value is accepted, so the caller trusts an algorithm choice that never happened) x reach=the only procedural-texture generator in the plugin, hit by any author making a non-photographic texture -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Three `create_noise_texture` calls differing only in `noiseType` (`Voronoi` / `Perlin` / `ZZZNotARealNoise`): the first two returned byte-identical content via `texture.get_pixel_stats` (`hash 5c4b7e25`, `mean.r 116.64346313476562`, `min 44`, `max 240`), and the third was created with no error or warning. Source-confirmed statically at HEAD in this tree: `TextureHandler.cpp:244` parses `noiseType` into a local that occurs nowhere else in the file except its `RPC_PARAM_DEF` at `:2517`; the pixel loop at `:275-311` calls `FBMNoise(...)` unconditionally on both branches. Capability gap confirmed separately: `material.graph.search_expression_types {query:"Voronoi"}` returns 0 results and the only cellular node is the per-pixel `MaterialExpressionVectorNoise`, so no cellular/Worley texture route exists at all. The correct rejection pattern already ships in `PCGAddNoiseFilter.cpp:135` (`"Unknown spatial noiseType: %s"`). Worked around on the Atlantis build with a Perlin base plus a V-shaped `texture.adjust_curves` master curve (recipe in the body); defect untouched.
