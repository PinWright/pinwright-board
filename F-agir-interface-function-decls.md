---
id: F-agir-interface-function-decls
title: "AGIR: round-trip anim layer interface declarations on the AnimBP"
status: DONE
severity: Medium
category: feature
tags: [agir, animgraph, roundtrip, interface, linked-anim, cliff]
---

# AGIR: round-trip anim layer interface declarations on the AnimBP

After `F-agir-state-alias` landed, the Lyra `ABP_Mannequin` round-trip surfaced the next cliff:

```
AGIR_TARGET_NOT_FOUND: no implemented interface declares function 'FullBodyAdditives'
[TestAnimGraphHandlers.cpp(658)]
```

`ABP_Mannequin` implements one or more anim layer interface assets (e.g. `ALI_ItemAnimLayers`, `ALI_LocomotionAnimLayers`) which declare layer functions like `FullBodyAdditives`, `UpperBodyAdditives`, etc. Decompile emits `linked_anim` / `linked_input_pose` blocks that reference these functions by name, but AGIR text doesn't carry the `Blueprint->ImplementedInterfaces[]` array, so the recompiled blueprint has no implemented interfaces and the compiler can't resolve which interface declares the referenced function.

The existing `linked_anim` and `linked_input_pose` opcodes (`F-agir-cliff-completion` items 4 and 5) work fine for synthetic fixtures where the test setup pre-populates `ImplementedInterfaces` before round-trip — the gap only shows on real-world AnimBPs where the interface set is part of the asset under test.

**Fix sketch:**

1. **Top-level metadata block** — add an `interfaces` declaration that lives at the root of the AGIR text (sibling of `anim_graph` blocks), e.g.:
   ```
   interfaces {
       implements `/Game/Characters/Heroes/Mannequin/Animations/ALI_ItemAnimLayers.ALI_ItemAnimLayers_C`
       implements `/Game/Characters/Heroes/Mannequin/Animations/ALI_LocomotionAnimLayers.ALI_LocomotionAnimLayers_C`
   }
   ```
   Sister to the existing `node_guid` / metadata-emit pattern. Decompile reads `Blueprint->ImplementedInterfaces[i].Interface` (a `TSubclassOf<UInterface>`) and emits one `implements` line per entry; compile populates `ImplementedInterfaces` and triggers `FBlueprintEditorUtils::ConformImplementedInterfaces` so the interface override graphs / function signatures get created.

2. **Or scope-by-text** — drop the `linked_anim.layer_name=...` reference and inline the interface class path on the `linked_anim` block itself, so each linked_anim self-describes its required interface. Less central but ties resolution failures to the offending block.

Approach (1) is closer to how UE itself models this (interface set is per-AnimBP, not per-node) and avoids per-node duplication. Approach (2) is simpler to add but smears interface metadata across every linked_anim block.

**Test:** `EditorAutomationRpcGateway.anim.agir.RoundTripStateMachine` (`TestAnimGraphHandlers.cpp:658`) is the load-bearing fixture. Should ship alongside a synthetic test that creates a minimal AnimBP implementing one interface with one layer function and exercises only this opcode.

## History
- `#1-initial-spec` `OPEN` reporter — Filed during `mcp-test-loop` cycle 4, immediately after `F-agir-state-alias` cleared the previous symbol-resolution failure. The Lyra mannequin round-trip is acting as a federated cliff detector: each landed feature reveals the next gap, exactly as `F-agir-cliff-completion#1-initial-spec` predicted ("recommend implementing alongside a synthetic-fixture regression test rather than relying on the Lyra mannequin AnimBP fixture").
- `#2-agir-interface-roundtrip` `IN-REVIEW` developer — Added top-level `interfaces { implements <classpath> }` AGIR block round-tripped by AGIROpcodes/Grammar/Parser/Decompiler/Compiler; compile-side pre-pass calls `FBlueprintEditorUtils::ImplementNewInterface` for each manifest entry before `BuildLayerGraphMap` runs. Regression test: `FAGIRInterfaceRoundTripTest` (`EditorAutomationRpcGateway.AGIR.Interface.RoundTrip`) at `Source/EditorAutomationRpcGatewayTests/Private/Assets/TestAGIRInterfaceRoundTrip.cpp`.
- `#3-verify-interfaces-block-emitted` `DONE` tester — Verified: `anim.decompile_agir` on `/Game/Characters/Heroes/Mannequin/Animations/ABP_Mannequin_Base` emits a top-level `interfaces { implements `/Game/Characters/Heroes/Mannequin/Animations/LinkedLayers/ALI_ItemAnimLayers.ALI_ItemAnimLayers_C` }` block as the first text section, ahead of the AnimGraph. Matches the round-trip spec; the linked_anim layers (`FullBodyAdditives`, `FullBody_Aiming`, etc.) referenced in the AnimGraph are now backed by the emitted interface manifest.
