---
id: B-niagara-scalability-not-captured
title: "Niagara compact model omits direct emitter scalability fields"
status: DONE
severity: Medium
category: bug
tags: [niagara, scalability, asset-dump, coverage-gap]
---

# Niagara compact model omits direct emitter scalability fields

Dump and RPC coverage exists for Niagara scalability. The remaining bug is limited to compact `niagara_model.json` placement for emitter scalability data.

Current compact model shape:
- System model root exposes system `scalability`.
- Nested `versionedEmitter.scalability` exists.
- System model `emitters[]` records omit direct `scalability`.
- Standalone emitter model roots omit direct `scalability`.

Consumers reading compact system emitter rows or standalone emitter roots should not have to descend into `versionedEmitter` to find emitter scalability. Keep the nested field for compatibility, but also expose a direct `scalability` object at both compact-model emitter surfaces.

## Fix

1. In `BuildEmitterHandleModel`, after `versionedEmitter`, add direct `scalability` from `NiagaraDumpBuilder::BuildEmitterScalabilityModel(EmitterData)`.
2. In `BuildEmitterModelJson`, after standalone `versionedEmitter`, add the same top-level direct `scalability` field.
3. Add a compact-model regression test that asserts `scalability.emitterScalability[0]` exists directly on both the system emitter record and standalone emitter model root.

## Repro

Build compact `niagara_model.json` for a Niagara system with an emitter scalability override and for a standalone Niagara emitter. The emitter scalability data is present under `versionedEmitter.scalability`, but missing from the direct emitter surfaces consumers expect:
- `emitters[0].scalability`
- root `scalability` in standalone emitter `niagara_model.json`

## History
- `#1-initial-spec` `OPEN` reporter — Niagara coverage parity audit found zero references to scalability fields in `NiagaraDumpBuilder.cpp`. Editor's Scalability panel data (system/emitter quality overrides, distance fade, platform sets, bOverride flags) is invisible to the IR. Required for cross-system performance budgeting / platform-targeted authoring.
- `#2-scalability-dump-and-set-rpc` `IN-REVIEW` developer — Added NiagaraDumpBuilder::BuildSystemScalabilityModel and BuildEmitterScalabilityModel (NiagaraDumpBuilder.cpp/.h) plus a file-local BuildPlatformSetJson helper. Wired scalability sections into BuildSystemJson, BuildEmitterAssetJson, and BuildEmittersJson; into BuildEmitterSummaryModel and BuildSystemModelJson on the model side. Added niagara.set_scalability_property RPC in NiagaraEditHandler.cpp targeting system or per-quality-level emitter overrides via the standard ApplyJsonValueToProperty path; calls UpdateScalability() after mutation. Regression test at TestNiagaraDumpScalability.cpp asserts the per-override JSON shape.
- `#3-returned-model-emitter-gap` `OPEN` tester — Returned: `asset.dump` on `/App/App/FXE_Trail` and `/Game/Effects/Particles/Environmental/Emitters/NE_BoneBasedEmission` shows scalability sections in `niagara_system.json` and `niagara_emitters.json`, and `niagara.set_scalability_property` is exposed, but the compact model is incomplete: the system model's emitter summary has no `scalability`, and the standalone emitter `niagara_model.json` has no scalability field.
- `#4-compact-model-scalability` `IN-REVIEW` developer — Added direct emitter scalability fields to system compact model emitter records and standalone emitter model roots using the existing NiagaraDumpBuilder scalability serializer; added FNiagaraModelBuilderEmitterScalabilityShapeTest to lock the compact model shape.
- `#5-verify-fix` `DONE` tester — Verified: `asset.dump` on `/App/App/FXE_Trail` shows direct `emitters[0].scalability` (keys: emitterScalability, platforms, present) alongside `versionedEmitter.scalability`; `asset.dump` on `/Game/Effects/Particles/Environmental/Emitters/NE_BoneBasedEmission` shows direct root `scalability` with the same shape as `versionedEmitter.scalability`. Empty `emitterScalability[]` arrays reflect that neither asset has authored per-quality overrides — structural parity (the actual fix) is in place at both compact-model surfaces.
