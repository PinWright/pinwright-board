---
id: B-worldpartition-mutations-false-success
title: "World Partition data-layer mutators report successful changes without verifying the engine outcome"
status: OPEN
severity: High
category: bug
tags: [world-partition, datalayer, mutation, false-success, readback]
encounters: 1
lastSeen: 2026-09-03T23:08:29+03:00
---

# World Partition data-layer mutations acknowledge changes the engine may reject

## What happens

`world_partition.set_datalayer` discards the boolean returned by
`UDataLayerEditorSubsystem::AddActorsToDataLayers`, then hardcodes
`added:true` and sends success
(`Source/PinWright/Private/Handlers/World/WorldPartitionHandler.cpp:257-269`).
In UE 5.8 that engine call returns `false` when the actor is invalid for the
layer or when `Actor->AddDataLayer` makes no change
(`C:/UE_5.8/Engine/Source/Editor/DataLayerEditor/Private/DataLayer/DataLayerEditorSubsystem.cpp:1034-1101`).
The response reads back actor identity, but never reads back layer membership.

The sibling `world_partition.cleanup_invalid_datalayers` collects instances
with missing assets, calls the void `DeleteDataLayer`, and increments
`DeletedCount` unconditionally before reporting that count as cleaned
(`WorldPartitionHandler.cpp:306-322`). The engine deletion returns early when
`CanBeRemoved()` is false and can also fail to remove the instance
(`DataLayerEditorSubsystem.cpp:1914-1932`); UE 5.8 external data-layer instances
always return false from `CanBeRemoved`
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Public/WorldPartition/DataLayer/ExternalDataLayerInstance.h:42`).

## Why it matters

Automation receives a green assignment or a positive deletion count even when
the requested membership did not change or invalid layers remain. It can save
the world, discard recovery context, and continue on a false premise. Severity
is High because these are direct authoring mutations whose normal response can
contradict authoritative engine state.

## What should happen

For `set_datalayer`, capture the engine boolean and independently read back that
the resolved actor contains the resolved layer; distinguish added,
already-present, and rejected outcomes instead of echoing `added:true`. For
cleanup, preflight removability, call the deletion, re-enumerate by stable
instance identity, and return explicit `deleted` and `failed` arrays/counts.
Return an error or a clearly partial result whenever the postcondition is not
satisfied.

## Workaround

After assignment, inspect the actor's data-layer membership through an
independent reader. After cleanup, enumerate the world's data layers again and
do not trust the reported count by itself.

## Related

- Catalog patterns `request-echo-not-result-readback` and
  `partial-nonatomic-success`.
- `B-create-datalayer-transient-asset-not-persistable` — sibling persistence
  defect for layer creation; it does not cover ignored mutation results.

## History

- `#1-filed-unverified-datalayer-results` `OPEN` reporter — Source-only pattern scan followed both handlers into UE 5.8 engine code and confirmed that the assignment return is discarded and cleanup counts attempted calls rather than observed removals. Board-wide dedup found no ticket for either outcome-check mechanism; the existing create-data-layer ticket concerns disk persistence. No build, test, editor, MCP call, or Saved-file access was performed.
