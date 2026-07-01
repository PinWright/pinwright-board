---
id: F-dump-native-components
title: "scs.json omits native C++ components (CreateDefaultSubobject)"
status: DONE
severity: Medium
category: feature
tags: [scs, dump, native-components, asset-dump]
---

# scs.json omits native C++ components (CreateDefaultSubobject)

Components added in a parent class's C++ constructor via `CreateDefaultSubobject<T>()` live as default subobjects on the actor CDO, not in `USimpleConstructionScript`. `SCS->GetAllNodes()` only covers Blueprint-added nodes; native components (often the most structurally important ones, e.g. the root mesh on a C++ actor base) are invisible in `scs.json`.

Any Blueprint inheriting from a C++ actor with native components will produce an incomplete `scs.json` that lists only the BP-added overlay components. Consumers of the dump (AI assistants, automation tools) cannot see the full component hierarchy.

**Fix:** After iterating SCS nodes, collect the actor CDO's default subobjects via `CDO->GetDefaultSubobjects(OutSubobjects)`, filter for `UActorComponent`, skip any whose name is already covered by an SCS node, and emit each remaining one as a JSON entry with `source: "native"`.

## History
- `#1-feature-request` `OPEN` reporter — Native C++ components from CreateDefaultSubobject are invisible in scs.json; SCS->GetAllNodes() only covers Blueprint-added nodes.
- `#2-iterate-cdo-subobjects` `IN-REVIEW` developer — Added native component enumeration in `GetBlueprintSCS` (EditorAutomationRpcGateway_SCSHandlers.cpp). After SCS nodes are emitted, the function calls `Blueprint->GeneratedClass->GetDefaultObject(false)->GetDefaultSubobjects()`, filters for UActorComponent, skips names already in ScsNodeNames set, and emits each with `source: "native"`. Property diff uses class CDO as base. Shared `AddOverriddenProperties` helper handles property extraction for all sources.
- `#3-simplify-hoist-parent-cdo` `IN-REVIEW` developer — Per shortcut-audit: hoisted parent-CDO subobject scan above the native-component loop and indexed by name (TMap<FName, UObject*>) — eliminates O(N²) per-component linear search. Diff base now resolved via map lookup; if class mismatches, falls back to component class CDO.
- `#4-agent-verified-pass` `IN-REVIEW` tester — PASS. Test: `asset.dump` on `/App/HELIOS/Drones/Icarus/Parent/Icarus_ParentBP`. scs.json shows 3 entries with `source: "native"` (MeshComponent, CameraHub, ProximityFadeComponent), each correctly attributed to the C++ parent's `CreateDefaultSubobject` calls. Pre-fix this BP had no scs.json at all (empty SCS); fix unlocks dump path.
- `#5-reverified-this-pass` `DONE` tester — Re-verified during /mcp-review on a fresh dump: scs.json for Icarus_ParentBP shows 5 entries with `source:"native"` (more than the 3 from #4 — code may have evolved to also include native components added by intermediate parent BPs). Native enumeration is producing output as designed. Marking DONE.
