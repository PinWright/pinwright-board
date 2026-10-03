---
id: B-instance-writers-bypass-foliage-ledger
title: "actor.set_instance_transforms and spatial.ground_instances write a foliage component's ISM instances directly, bypassing the FFoliageInfo ledger, so the edit is silently reverted the next time the foliage system rebuilds the component from the ledger"
status: OPEN
severity: Medium
category: bug
tags: [actor, spatial, ism, hism, foliage, instanced-foliage-actor, ledger, silent-revert, set_instance_transforms, ground_instances]
encounters: 1
lastSeen: 2026-10-03T03:40:00-05:00
rice: [1, 3, 0.8, 2]
priority: 13
---

# The typed instance writers accept a foliage component and edit only half of it

`InstancedMeshUtils::ResolveInstancedComponent` (`Handlers/Actor/InstancedMeshUtils.h`) enumerates
every `UInstancedStaticMeshComponent` on the actor, and `UFoliageInstancedStaticMeshComponent`
derives from it. So `actor.set_instance_transforms` and `spatial.ground_instances` accept an
`AInstancedFoliageActor` (with one foliage type the component is picked without being named) and
write through `UpdateInstanceTransform` on the component only.

Foliage keeps its own per-instance ledger, `FFoliageInfo::Instances` (location, rotation,
scale per instance), and the component is derived from it. `FFoliageInfo::ReallocateClusters`
(`Runtime/Foliage/Private/InstancedFoliage.cpp:2650-2680`) uninitializes the implementation and
re-adds every instance from the ledger, so a component-only move is silently reverted on the next
reallocation (a foliage-type settings change, among others). Foliage-mode selection and move tools
also read the ledger, so until then the two disagree. The verbs report `updated`/`placed` measured
off the component, which is correct at the time and wrong afterwards.

Expected: refuse a foliage component (`Actor->IsA<AInstancedFoliageActor>()` or
`Component->IsA<UFoliageInstancedStaticMeshComponent>()`) with `INVALID_TARGET_KIND`, naming the
`foliage.*` verbs, or route the write through `FFoliageInfo`. A count-changing write
(`actor.add_instances` / `actor.remove_instances`, `F-ism-create-and-clear-scatter`) is worse: it
breaks the 1:1 ledger/component index mapping, and `FFoliageInfo::CheckValid` asserts
`Instances.Num() == Implementation->GetInstanceCount()` (`InstancedFoliage.cpp:2200`, under
`DO_FOLIAGE_CHECK`). That half is raised against those new verbs in their review.

Related: `B-foliage-remove-empties-ledger-not-component` (the mirror image: ledger edited,
component not).

## History
- `#1-writers-skip-foliage-ledger` `OPEN` reporter — Found while reviewing
  `F-ism-create-and-clear-scatter`. Source-read only, not reproduced live: the resolver accepts
  `UFoliageInstancedStaticMeshComponent`, neither writer checks for it, and
  `FFoliageInfo::ReallocateClusters` rebuilds the component from `FFoliageInfo::Instances`
  (`InstancedFoliage.cpp:2658-2680`). Dedupe: grepped the board for `ledger`, `FFoliageInfo`,
  `InstancedFoliageActor` together with `set_instance_transforms` / `ground_instances`; only
  `B-ground-instances-default-component-foreign-scatter` mentions the ledger, and only as a
  compounding effect of `foliage.remove`.
