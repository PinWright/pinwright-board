---
id: F-rpc-property-omit-oversized-opt-in
title: "Add `omitOversized` opt-in to property.get and property.list"
status: DONE
severity: Low
category: feature
tags: [property, asset-dump-parity, rpc, oversized]
---

# Add `omitOversized` opt-in to property.get and property.list

`PropertyUtils::IsKnownOversizedProperty` + `BuildOmissionPlaceholder` (PropertyUtils.cpp:1812-1884) maintain a small allow-list of UPROPERTYs that are known to balloon JSON output: `AudioImpulseResponse::ImpulseResponse` (TArray<float> audio samples), `BodySetup::AggGeom` (physics geometry), `InstancedStaticMeshComponent::PerInstanceSMData` (per-instance transforms). When detected, the dumpers emit a `{ "$omitted": "...", "$reason": "exceeds-llm-budget" }` placeholder instead of the full payload.

Today this allow-list is consulted only by dump-side code paths: `EditorAutomationRpcGateway_SCSHandlers.cpp:179-185` (SCS template diff) and `PropertyUtils.cpp:2055` (CDO property export, used by `asset.dump`). The live generic readers — `property.get` (UtilityPropertyHandler.cpp:1038) and `property.list` (UtilityPropertyHandler.cpp:1182) — call `ExportPropertyToJsonValue` directly with no oversized check, so reading e.g. `PerInstanceSMData` on a foliage ISM component via `property.get` will fully serialize the array. On large levels this risks exceeding the 1MB request body limit, exhausting LLM context, or hanging the editor on the JSON walk.

This is the inverse of the asset-dump-parity policy in `F-rpc-niagara-inspect-standalone-script`: dump-only behavior should also be reachable from live RPCs. Here the dump shrinks, the live RPC does not — agents debugging a foliage component or audio asset have no way to ask the live reader for the same shrunk view.

`container.array.get` is intentionally excluded — it's per-element by index and cannot accidentally dump a mega-array. `container.array.append` / `set` / `clear` are writers and irrelevant. Scope is `property.get` and `property.list` only.

**Fix:** Add an opt-in `omitOversized` boolean param (default `false` to preserve current behavior) on `property.get` and `property.list`. When true, the handlers consult `IsKnownOversizedProperty` before calling `ExportPropertyToJsonValue` and emit the same `BuildOmissionPlaceholder` shape the dumpers already emit. Update `Docs/wiki/property.md` to document the new param. Opt-in (not opt-out) because existing callers may depend on full serialization, and the allow-list is small enough that callers who need it know to pass the flag.

## History
- `#1-initial-repro` `OPEN` reporter — `PropertyUtils::IsKnownOversizedProperty` / `BuildOmissionPlaceholder` are consulted only by SCS dump (EditorAutomationRpcGateway_SCSHandlers.cpp:179-185) and CDO dump (PropertyUtils.cpp:2055); live `property.get` (UtilityPropertyHandler.cpp:1038) and `property.list` (UtilityPropertyHandler.cpp:1182) skip the allow-list and fully serialize known-huge arrays (e.g. PerInstanceSMData, ImpulseResponse, AggGeom), risking 1MB body limit overruns and editor hangs. Fix: add opt-in `omitOversized` boolean (default false) on both methods that emits the existing dump placeholder shape. Excludes `container.array.get` (per-element, naturally bounded) and the array writers.
- `#2-omit-oversized-live-readers` `IN-REVIEW` developer — Added opt-in omitOversized handling to property.get and property.list using the existing oversized placeholder helper; documented the parameter and added utility handler regression coverage.
- `#3-verify-fix` `DONE` tester — Verified: `call("property.get")` on `/Script/Engine.Default__InstancedStaticMeshComponent` propertyName `PerInstanceSMData` with `omitOversized:true` returned `{"$omitted":"per-instance transforms, 0 elements","$reason":"exceeds-llm-budget","type":"TArray<FInstancedStaticMeshInstanceData>"}`; without the flag the same call returned `value: []` (full serialization). Wiki page for `property.get` and `property.list` both list the new `omitOversized` param.
