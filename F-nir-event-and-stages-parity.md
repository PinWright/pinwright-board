---
id: F-nir-event-and-stages-parity
title: "NIR: add eventHandlers, simulationStages, and gpuScript distinction"
status: DONE
severity: Medium
category: feature
tags: [niagara, nir, dump-format, parity]
---

# NIR: event handlers, simulation stages, GPU script parity with niagara_model.json

Blocker for [F-remove-niagara-json-sidecars](F-remove-niagara-json-sidecars.md). NIR has zero coverage of three Niagara features that `niagara_model.json` carries under `advancedFeatures`.

## Gaps

1. **`eventHandlers[]`** — event-driven particle reactions (e.g. collision → spawn child particles, death event). Today: 0 NIR lines. Some emitters use this heavily.

   Proposed NIR syntax:
   ```
   emitter CollisionFx enabled {
       ...
       eventHandler "OnCollide" {
           source:      Particles.PrimaryEmitter
           reaction:    SpawnBurst
           maxPerFrame: 100
       }
   }
   ```

2. **`simulationStages[]`** — custom simulation stages (post-spawn, post-update, custom data-interface compute). Used for advanced physics and per-frame computation. Today: 0 NIR lines.

   ```
   emitter Name enabled {
       simulationStage "Apply Field" {
           iterations:   2
           bSpawnOnly:   false
           dataInterface: Particles.NDIGrid3D
       }
   }
   ```

3. **`gpuScript` vs CPU script distinction.** Today NIR only shows `simTarget: CPUSim/GPUComputeSim` at the emitter level; doesn't link to the actual GPU compute graph content. When an emitter is GPU-sim, the GPU graph is a separate script with its own modules — NIR conflates this with the regular emitter scripts.

   ```
   emitter GPUParticles enabled {
       simTarget: GPUComputeSim
       gpuScript {
           graph: NiagaraGraph_GPU_0
           ...
       }
   }
   ```

## Implementation

Extend `NIRTextEmitter.cpp` to walk `advancedFeatures.eventHandlers`, `advancedFeatures.simulationStages`, and `advancedFeatures.gpuScript` from the source `UNiagaraEmitter` and emit structured sections.

Estimate: ~1 week. Mostly additive emission; requires identifying the right UE source API for each (likely `UNiagaraEmitter::GetEventHandlers()` and similar).

## Acceptance criteria

- For each sample asset containing event handlers, NIR shows a non-empty `eventHandler` block per JSON entry.
- For each asset containing simulation stages, NIR shows a non-empty `simulationStage` block per JSON entry.
- For GPU emitters, NIR distinguishes the GPU graph content from CPU-side emitter scripts.

## Implementation notes

Implemented in `Private/NIR/NIRDecompiler.cpp` using script-graph emission tokens from `Private/NIR/NIRTextEmitter.cpp`.

Static review boundary:
- `Private/NIR/NIRDecompiler.cpp` emits `eventHandler` blocks, preserves the `simStage` block keyword, and includes GPU compute script graph content via `ParticleGPUCompute`.
- `Private/Tests/Niagara/TestNIRDecompiler.cpp` asserts event-handler text, simulation-stage text, and GPU compute graph text.
- No Unreal build or runtime test has been run for this board transition.

## History
- `#1-initial-parity-audit` `OPEN` reporter — NIR has 0 lines for eventHandlers + simulationStages + gpuScript distinction. Blocker for removing niagara_model.json. Effort ~1 week.
- `#2-verify-fix` `DONE` tester — Verified: ran `asset.dump` on `/Game/SoStylized/Effects/NS_MeteorShower.NS_MeteorShower` (an asset whose `niagara_model.json` shows non-empty `eventHandlers`). Fresh `nir.txt` contains live `eventHandler LocationEvent @0 {...}` blocks emitting full event-handler reflected fields (ExecutionMode, MaxEventsPerFrame, MinSpawnNumber, SourceEmitterID, SourceEventName, SpawnNumber, UpdateAttributeInitialValues) plus their own `graph ParticleEvent` scope, and `graph ParticleGPUCompute {...}` scopes alongside the CPU graphs for GPU-script content. `simStage` emission code path present in `NIRDecompiler.cpp::AppendSimStages` but no project asset carries non-empty `simulationStages` to exercise it live; the two observable acceptance criteria pass.
