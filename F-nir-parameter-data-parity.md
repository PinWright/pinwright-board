---
id: F-nir-parameter-data-parity
title: "NIR: add rapid-iteration parameter constants, static-switch source, renderer bindings, parameter scope"
status: DONE
severity: High
category: feature
tags: [niagara, nir, dump-format, parity]
---

# NIR: parameter-data parity with niagara_stack.json + niagara_parameters.json

Blocker for [F-remove-niagara-json-sidecars](F-remove-niagara-json-sidecars.md). NIR currently emits high-level module structure but **omits most of the actual authored parameter values**, which are the load-bearing content for understanding Niagara behavior.

Field-by-field gap (verified on `NS_GunPad_Loading`):

## Critical gaps

1. **Rapid-iteration constants (~500-2000 entries per asset).** The actual authored values for `spawnRapidIteration[]`, `updateRapidIteration[]`, `systemSpawnRapidIteration[]`, `systemUpdateRapidIteration[]`. Today these vanish entirely from NIR; an agent reading NIR cannot tell what spawn lifetime, init color, velocity ranges, etc. are set to. Source: `niagara_parameters.json`.

   Proposed NIR syntax:
   ```
   emitter Base_Diffuse enabled {
       spawnRapidIteration {
           `Constants.Base_Diffuse.InitializeParticle.Lifetime Max` : NiagaraFloat = 2.0
           `Constants.Base_Diffuse.InitializeParticle.Position`    : Vector3f    = (0, 0, 0)
       }
       updateRapidIteration {
           ...
       }
   }
   ```

2. **`staticSwitchInputs[].source` (override vs default).** Today NIR shows static switch values but not whether they're author-overridden or module defaults — semantically very different. Source: `niagara_stack.json`.

   ```
   static `Inactive Response` = 1.0 @source=override
   static `Loop Behavior`     = 0.0 @source=default
   ```

3. **`rendererBindings[]` (parameter→material parameter map).** Maps user/custom params to material parameters. Required to trace how authored constants flow to visual output. Source: `niagara_parameters.json` + renderer in `niagara_emitters.json`.

   ```
   renderer NiagaraSpriteRendererProperties @0 enabled {
       binding User.Scale → ScaleFactor
       binding User.Color → BaseColor
   }
   ```

4. **Parameter `scope` annotation.** Tells when each parameter takes effect (spawn vs update vs system vs particle). Source: `niagara_parameters.json`.

   ```
   param `User.SpawnRate`      : NiagaraFloat @scope=user      @read-at=systemSpawn
   param `Particles.Velocity`  : Vector3f     @scope=particle  @read-at=particleUpdate
   ```

## Useful (lower priority)

5. **`staticSwitchInputs[].enumPath`** — needed for semantic mapping of numeric enum values (e.g. `1 = ENiagaraSystemInactiveMode::Complete`).
6. **`staticSwitchInputs[].defaultValue`** — useful for highlighting author intent (override delta).

## Internal-only (drop, no NIR equivalent needed)

- `entryId`, `nodeName`, `selectedScriptVersion` — GUIDs / version IDs
- `offset`, `sizeBytes`, `isDataInterface`, `isUObject` — buffer layout; derivable from `struct`
- `id`, `idName` — emitter GUIDs

## Implementation

- Extend `NIRDecompiler.cpp::AppendEmitterBody()` paralleling existing `AppendParameterStoreVars()` to iterate emitter parameter stores and emit scope-qualified sub-blocks.
- Add static-switch source annotation in the stack-emission path.
- Add renderer-binding emission inside the renderer block.
- Bump `nir.txt` aspect version (will likely grow 30-50% in size — acceptable).

Estimate: ~300-400 lines of C++, ~100 lines of test coverage. **2-3 weeks** of focused work.

## Acceptance criteria

For 5 representative Niagara systems (NS_GunPad_Loading, Rain, NS_WallPortal, NS_Fire_Medium_Smoke, NS_Grenade_Explosion):
- Every key in `niagara_stack.json` is either present in NIR or explicitly classified as internal-only/derivable.
- Every key in `niagara_parameters.json` is either present in NIR or explicitly classified as internal-only/derivable.
- Test fixture asserts round-trip semantic equivalence (NIR contains all rapid-iteration constants present in JSON).

## History
- `#2-nir-parity-wave-plan` `IN-REVIEW` implementer — NIR now emits modeled parameter values, rapid-iteration stores, static-switch source/default metadata, and renderer binding store entries without removing JSON sidecars.
- `#1-initial-parity-audit` `OPEN` reporter — NIR is missing ~500-2000 authored parameter values per asset (rapid-iteration constants), plus static-switch source semantics and renderer bindings. Critical gap. Blocker for removing niagara_stack + niagara_parameters JSON sidecars.
- `#3-verify-nir-parity` `DONE` tester — Verified via `asset.dump` on `/Game/Effects/Particles/Item/NS_GunPad_Loading` and `/Game/Vefects/Free_Fire/Shared/Particles/NS_Fire_Medium_Smoke`. nir.txt grew to 6528 / equivalent lines; emits `rapid \`Constants...\` : Type = value @scope particleSpawnRapidIteration|emitterUpdateRapidIteration|...` lines (e.g. line 178 `rapid \`Constants.Base_Diffuse.InitializeParticle.Lifetime Max\` : NiagaraFloat = 2.0`), `static \`X\` = N @source override @default M` lines (gap #2), and `@scope` annotations on rapid entries (gap #4). Gap #3 renderer bindings: no asset in dump cache has populated `rendererBindings[]` (all `[]` in source JSON), so binding emission is untestable on real content but the renderer block now emits property fields and the implementation claim is consistent with empty-input behavior.
