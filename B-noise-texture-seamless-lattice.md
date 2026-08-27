---
id: B-noise-texture-seamless-lattice
title: "`texture.create_noise_texture {seamless:true}` collapses its 4 torus coordinates into 2 by addition, so the only tiling-safe path emits a periodic diagonal lattice instead of noise"
status: OPEN
severity: High
category: bug
tags: [texture, create_noise_texture, noise, seamless, tiling, wrong-pixels, silent-noop]
encounters: 1
lastSeen: 2026-08-27T18:47:15+05:00
---

# `texture.create_noise_texture {seamless:true}` produces a woven diagonal lattice, not tileable noise — the seamless path collapses a 4D torus into 2D by summing coordinate pairs

With `seamless:true` the output is not noise. It is a regular diagonal weave:
one motif repeated on a grid, with hard diagonal streaks running through it, at
every `scale` and `octaves` setting tried. With `seamless:false` the identical
parameters — same seed, same everything, one flag apart — give correct, organic
Perlin FBM.

The call reports `success` either way. Nothing in the response says the image is
degenerate, and there is no other tiling-safe option in the namespace, so every
procedural texture an author ships either carries a wrap seam or carries this
lattice.

## Root cause (guilty source line)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureHandler.cpp:282-292`:

```cpp
    // Seamless tiling using domain wrapping
    float Angle1 = NX * PI * 2.0f;
    float Angle2 = NY * PI * 2.0f;
    float NX3D = FMath::Cos(Angle1);
    float NY3D = FMath::Sin(Angle1);
    float NZ3D = FMath::Cos(Angle2);
    float NW3D = FMath::Sin(Angle2);
    NoiseValue = FBMNoise(NX3D + NZ3D, NY3D + NW3D, Octaves, Persistence, Lacunarity, Seed);
```

The standard 4D-torus trick for seamless noise requires a **4D** noise function:
you map (u,v) onto a torus in 4-space and sample 4D noise there. This code builds
the four torus coordinates correctly and then **collapses them into two by
addition** before feeding a **2D** `FBMNoise`.

`cos(a) + cos(b)` and `sin(a) + sin(b)` are symmetric in their arguments and
non-injective over the unit square: distinct (X, Y) map to identical sample
points along the `X+Y` and `X-Y` diagonals. That degeneracy is exactly what the
output shows — a single motif repeated on a lattice with hard diagonal bands
running through it. The comment on `:282` claims "domain wrapping"; no wrapping
occurs.

`Seed` is passed through untouched on both branches (`:292` seamless, `:296`
non-seamless), which is consistent with the measured finding below that `seed`
is honoured in the seamless path — the fault is confined to the coordinate
collapse, not to the generator.

## Verbatim repro

Same seed, same everything, one flag apart:

```
texture.create_noise_texture {name:"S", width:512, height:512, scale:3, octaves:5,
                              persistence:0.62, lacunarity:2.1, seed:909, seamless:true}
texture.create_noise_texture {name:"N", width:512, height:512, scale:3, octaves:5,
                              persistence:0.62, lacunarity:2.1, seed:909, seamless:false}
```

Thumbnails via `asset.generate_thumbnail`, both 512x512, measured 2026-08-27 on
the Atlantis build:

- `seamless:true`, scale 3 (`T_Stone_Grain.png`) — a 4x4 grid of an identical
  ring-shaped motif with hard diagonal bands crossing it. Reads as woven fabric.
- `seamless:false`, same seed and scale (`T_TMP_NoSeam.png`) — correct cloudy
  FBM, no repetition, no banding.
- `seamless:true`, scale 26 (`T_Sand_Grain.png`) — a fine regular grid over flat
  grey. Global range is fine (`min 54`, `max 199` on a re-run), so this is not a
  contrast or normalisation fault: the *structure* is periodic at the lattice, so
  it reads as a screen door rather than grain.

**Control that isolates it to the tiling path, not the generator:** `seed` **is**
honoured with `seamless:true` — scale 4 / octaves 3 / seamless, seed 1 vs 99999
gave hashes `6e47dee2` and `c1dafc4f`. So the seamless branch is running the
noise function and threading the seed; it is feeding it degenerate coordinates.

## What it should do

Sample genuine 4D noise at `(NX3D, NY3D, NZ3D, NW3D)` rather than summing pairs
into a 2D lookup. If a 4D noise function is not available, the honest
alternatives are (a) generate at 2x and mirror-fold, or (b) refuse
`seamless:true` with a typed error rather than emit a lattice and report success.
Emitting a degenerate image under a flag whose entire purpose is correctness is
the worst of the three.

## Impact

The only tiling-safe option produces output no artist would ship, so every
procedural texture on the Atlantis build is `seamless:false` and carries a wrap
seam. On a 24000 x 24000 seafloor that seam is a straight line across the
terrain.

## Workaround

`seamless:false`, then break up the wrap in the material by sampling the same
texture at two incommensurate tiling rates and combining them — `M_Seafloor`
samples `T_Sand_Grain` at `GrainScale` and `T_Stone_Grain` at `MottleScale`,
roughly 20x apart. Heavy fog does the rest. This mitigates a visible seam; it
does not make the texture tile.

## Distinct from related tickets

- `B-noise-texture-noisetype-ignored` is the sibling defect in the same verb,
  found in the same session: `noiseType` is parsed into a dead local at
  `TextureHandler.cpp:244` and never branched on. Different guilty lines
  (`:284-292` vs `:244`), same handler — fix them together while the file is
  open.
- `F-texture-sampling-settings-batch` (OPEN, Low) concerns wrap/filter/group/LOD
  **addressing mode** for a tiling material. Word collision with "tiling" only;
  nothing to do with what the generator draws.
- `B-texture-save-no-disk-write` (IN-REVIEW, Critical) is about `save:true`
  marking dirty without writing bytes. Orthogonal — these textures reached disk
  and read back exactly the wrong pixels they were generated with.

severity rationale: impact=silent wrong output on a normal path (the flag whose only purpose is correctness produces a degenerate image and reports success) x reach=the only procedural-texture generator in the plugin, and the only tiling-safe setting it offers -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. One-flag-apart repro at seed 909 / scale 3 / octaves 5: `seamless:true` gives a 4x4 grid of one ring motif with hard diagonal bands, `seamless:false` gives correct cloudy FBM. Reproduced at scale 26 as a fine regular grid over flat grey with a healthy global range (`min 54`, `max 199`), so it is structural, not tonal. Isolating control: `seed` IS honoured on the seamless path (seed 1 vs 99999 → hashes `6e47dee2` / `c1dafc4f`), pinning the fault to the tiling branch rather than the generator. Source-confirmed statically at HEAD in this tree: `TextureHandler.cpp:282-292` builds the four torus coordinates `cos/sin(2*pi*NX)` and `cos/sin(2*pi*NY)`, then sums them pairwise into two arguments for a **2D** `FBMNoise` — `cos(a)+cos(b)` / `sin(a)+sin(b)` are symmetric and non-injective, so distinct (X,Y) collapse onto the `X±Y` diagonals, which is precisely the observed banding and motif repetition. The comment on `:282` claims "domain wrapping"; none occurs. Worked around on the Atlantis build by shipping `seamless:false` and dual-scale sampling in `M_Seafloor`; defect untouched.
