---
id: F-nir-emitter-flags-parity
title: "NIR: add localSpace, fixedBounds, GPU allocation hints, warmup, determinism, scalability"
status: DONE
severity: Medium
category: feature
tags: [niagara, nir, dump-format, parity]
---

# NIR: emitter+system flag parity with niagara_emitters.json + niagara_system.json

Blocker for [F-remove-niagara-json-sidecars](F-remove-niagara-json-sidecars.md). NIR is missing system- and emitter-level flags that affect spatial, temporal, and performance behavior.

Field gaps (verified on `NS_GunPad_Loading` + `Rain`):

## Critical

1. **`localSpace` (emitter spatial mode)** — whether particles spawn in emitter or world space. Essential for understanding spatial behavior. Today: absent.

   ```
   emitter Base_Diffuse enabled {
       localSpace: true
       simTarget:  CPUSim
       ...
   }
   ```

2. **`fixedBounds` (emitter + system level)** — explicit bounds override; critical for LOD / culling / performance analysis. Today: absent.

   ```
   system "..." {
       fixedBounds { min: (-100, -100, -10), max: (100, 100, 50) }
   }
   emitter Name { fixedBounds { ... } }
   ```

## Useful

3. **GPU allocation hints** — `maxGpuParticlesSpawnPerFrame`, `preAllocationCount`.

   ```
   emitter Name enabled {
       maxGpuParticlesSpawnPerFrame: 1024
       preAllocationCount:           512
   }
   ```

4. **Warmup settings** — `warmupTickCount`, `warmupTickDelta`, `warmupTime`.

   ```
   system "..." {
       warmupTicks: 120
       warmupDelta: 0.0667
   }
   ```

5. **`determinism` flag** — at system + emitter scope; affects reproducibility analysis.

6. **`scalability.systemScalability[]`** — per-platform quality settings.

## Internal-only (drop)

- `id`, `idName`, `usageId`, `graphPath` — GUIDs/internal references
- `deprecated`, `deprecationMessage` — legacy flags (could surface if a sweep reports deprecation issues)
- `readyToRun`, `needsWarmup`, `needsRecompile` — editor/runtime state
- `emitterCount` — derivable from `emitterHandles[]` length

## Implementation

Extend `NIRTextEmitter.cpp` to emit the additional fields at system and per-emitter scope. No new structural changes — purely additive lines.

Estimate: ~1 week for Critical (`localSpace`, `fixedBounds`); +1 week for Useful (GPU hints, warmup, scalability, determinism).

## Acceptance criteria

Same 5 representative Niagara systems as `F-nir-parameter-data-parity`:
- All emitter/system keys present in NIR or classified as internal/derivable.
- Test fixture asserts emit-and-parse for the new fields.

## Implementation notes

Implemented in `Private/NIR/NIRDecompiler.cpp` with source parity against `Private/Handlers/Niagara/NiagaraDumpBuilder.cpp`.

Static review boundary:
- `Private/NIR/NIRDecompiler.cpp` emits system warmup/fixed-tick, determinism/random seed, fixed bounds, scalability, emitter handle state, and emitter local-space/bounds/GPU allocation/preallocation flags.
- `Private/Tests/Niagara/TestNIRDecompiler.cpp` asserts representative fixed-bounds, local-space, GPU spawn cap, and preallocation NIR text.
- No Unreal build or runtime test has been run for this board transition.

## History
- `#1-initial-parity-audit` `OPEN` reporter — NIR misses spatial (`localSpace`), bounds (`fixedBounds`), performance hints (GPU alloc, warmup), and reproducibility (`determinism`) flags. Required to remove niagara_emitters + niagara_system sidecars.
- `#2-verify-fix` `DONE` tester — Verified: `asset.dump` on `/Game/Effects/Particles/Item/NS_GunPad_Loading` wrote `nir.txt` containing system-scope `warmupTime`/`warmupTickCount`/`warmupTickDelta`/`determinism`/`randomSeed`/`fixedBoundsEnabled`/`fixedBounds`/`scalability` and per-emitter `localSpace`/`determinism`/`fixedBounds`/`maxGpuParticlesSpawnPerFrame`/`preAllocationCount`/`scalability`/`simTarget` — all six ticket categories present.
