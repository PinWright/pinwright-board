---
id: B-worldpartition-mutations-false-success
title: "World Partition data-layer mutators report successful changes without verifying the engine outcome"
status: IN-REVIEW
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

## Fix

Root cause: `set_datalayer` discarded `AddActorsToDataLayers`' engine result and
reported `added:true` without checking actor membership. Cleanup counted every
`DeleteDataLayer` call even though the engine API is void and may reject an
instance before removal. The initial correction still treated already-present
membership and an empty cleanup scan as successful even though neither changed
engine state.

Changed files:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/ErrorCodes.h` — register narrow typed errors for duplicate assignment and cleanup with no invalid candidates.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/World/WorldPartitionHandler.cpp` — capture and expose assignment outcomes with actor membership readback; refuse duplicate/no-change assignment; preflight cleanup removability, re-enumerate by stable instance path, and return verified deleted/failed rows with typed failure codes, including an empty scan.
- `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestWorldPartitionDataLayerMutations.cpp` — a transient `EWorldType::Editor` World Partition fixture is made active only for each test, invokes both production handlers through `InvokeHandlerWithCapture`, verifies duplicate and ineligible-actor assignment error payloads plus zero-candidate and mixed cleanup results, and restores the original editor world before scoped destruction/discard; the direct engine mutation/readback coverage remains.
- `Plugins/PinWright/Docs/wiki-src/world_partition.md` — document the readback and typed no-change failure contract.

Test IDs:

- `PinWright.world_partition.mutations.TransientWorldDataLayerReadback`
- `PinWright.world_partition.set_datalayer.ReadbackContract`
- `PinWright.world_partition.cleanup_invalid_datalayers.ReadbackContract`

Deliberate non-changes: no engine source, Config, Saved, PIE/live-editor map
fixture, build/test run, MCP call, unrelated World Partition verbs, or outer PDS
files were changed. The public engine surface reliably forces the real
`bEngineChanged:false` / absent-membership refusal with a non-external actor;
the narrower contradictory `bEngineChanged:true` / absent-membership state
cannot be induced without an engine seam, so the test does not stub or fabricate
it. Live engine verification remains for the tester.

## History

- `#1-filed-unverified-datalayer-results` `OPEN` reporter — Source-only pattern scan followed both handlers into UE 5.8 engine code and confirmed that the assignment return is discarded and cleanup counts attempted calls rather than observed removals. Board-wide dedup found no ticket for either outcome-check mechanism; the existing create-data-layer ticket concerns disk persistence. No build, test, editor, MCP call, or Saved-file access was performed.
- `#2-verified-worldpartition-mutation-readback` `IN-REVIEW` developer — Implemented engine-result capture, actor membership readback, cleanup removability preflight, stable-path re-enumeration, explicit outcome arrays/counts, and typed verification/deletion errors. Added the two structural automation IDs listed above and documented the response contract. Static inspection only; no tests, builds, editor, PIE, MCP, or Saved-file access was performed.
- `#3-transient-world-readback-coverage` `IN-REVIEW` developer — Replaced the sole source-scan coverage with a GUID-named transient World Partition fixture that creates a data layer, exercises actor assignment/removal and duplicate no-op behavior, reads actor membership back, removes the layer, and re-enumerates the layer collection by stable path under `FScopedTransientWorldGuard`. Retained the handler source checks as secondary wiring coverage. Static inspection only; no tests, builds, editor, PIE, MCP, or Saved-file access was performed.
- `#4-transient-subsystem-readback` `IN-REVIEW` developer — Corrected the fixture to use an unregistered `EWorldType::Editor` transient world (required by UE 5.8 subsystem eligibility) and the exact `UDataLayerEditorSubsystem::AddActorsToDataLayers` / `RemoveActorsFromDataLayers` calls, including duplicate no-op and actor/layer manager readbacks. Test IDs are unchanged. Static inspection only; no tests, builds, editor, PIE, MCP, or Saved-file access was performed.
- `#5-delete-api-readback-fixture` `IN-REVIEW` developer — Corrected the cleanup fixture to call the same void `UDataLayerEditorSubsystem::DeleteDataLayer` API as production, then resolve the captured instance name and re-enumerate by its captured path without dereferencing the removed UObject. Test IDs are unchanged. Static inspection only; no tests, builds, editor, PIE, MCP, or Saved-file access was performed.
- `#6-transient-world-safe-teardown` `IN-REVIEW` developer — Added ordered scoped teardown for the never-saved partitioned world: `FScopedTransientWorldGuard` destroys it first, then the shared asset-discard helper detaches the world object and collects its package. Test IDs are unchanged. Static inspection only; no tests, builds, editor, PIE, MCP, or Saved-file access was performed.
- `#7-typed-no-change-refusal` `IN-REVIEW` developer — Corrected the response contract so an already-present assignment returns `DATALAYER_ALREADY_ASSIGNED`, every other engine-reported assignment no-change returns `VERIFICATION_FAILED`, and cleanup with no invalid candidates returns `NO_INVALID_DATALAYERS`; all errors retain their observed result payload. Strengthened the existing structural contract IDs while preserving the transient-world duplicate no-op fixture. Static inspection only; no tests, builds, editor, PIE, MCP, git mutation, or Saved-file access was performed.
- `#8-handler-level-transient-coverage` `IN-REVIEW` developer — Replaced the two handler source scans with real `InvokeHandlerWithCapture` calls against the transient partitioned editor world. The set test verifies `DATALAYER_ALREADY_ASSIGNED` and its observed duplicate payload, then forces UE's real actor-eligibility refusal with `bCreateActorPackage=false` and verifies `VERIFICATION_FAILED` plus the rejected actor payload. The cleanup test verifies `NO_INVALID_DATALAYERS`, then combines a removable missing-asset base instance with an engine-unremovable missing-asset external instance to verify `DELETE_PARTIAL`, the deleted/failed entries, and fresh enumeration. Test IDs are unchanged. Static inspection only; no tests, builds, editor, PIE, MCP, git mutation, or Saved-file access was performed.
