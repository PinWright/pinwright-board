---
id: B-ism-undo-record-unsafe
title: "ISM per-instance undo record is not self-describing — a local-space movedInstances[] replayed at the default destroys the scatter and reports success"
status: OPEN
severity: High
category: bug
tags: [actor, spatial, ism, hism, instanced-static-mesh, per-instance, undo, movedInstances, previousTransform, silent-wrong-write, silent-noop, transaction, contract]
encounters: 1
lastSeen: 2026-08-29
---

# The ISM undo record has no invariants, and the reason it exists is arithmetically wrong

`F-ism-per-instance-transforms` shipped three verbs — `actor.get_instances`,
`actor.set_instance_transforms` (`Handlers/Actor/InstancedMeshHandler.cpp`) and
`spatial.ground_instances` (`Handlers/Spatial/GroundPlacementHandler.cpp`) — that deliberately
decline `FScopedTransaction` and substitute an echoed pre-write pose, `movedInstances[]`, as **the**
undo mechanism. The rationale is recorded in the header comment of
`Handlers/Actor/InstancedMeshUtils.h`.

Two independent problems, pointing opposite ways:

1. **The record is not a contract.** It is an array of `{index, previousTransform}` with no statement
   of what those numbers mean. Whether it is replay-safe depends on facts held only in the
   surrounding response, or in no field at all. One replay path silently destroys a scatter and
   answers `updated: N`.
2. **The stated reason for having no transaction does not survive reading the engine.** The
   O(written x total) undo-record figure assumes each `Modify()` inside a transaction appends a fresh
   snapshot. `FTransaction::SaveObject` deduplicates per object per transaction, so one
   `FScopedTransaction` around the whole batch costs exactly **one** snapshot — a few MB on a
   10k-instance scatter, once. The design traded away working Ctrl+Z for a cost that does not exist.

**This ticket recommends replacing the design, not patching five holes in it:** wrap the batch in one
transaction — which is what every other mutating verb in this plugin already does (183
`FScopedTransaction` sites across ~40 handler files; these three verbs are the only opt-outs) — and
*keep* the record, made self-describing, for what it can do that a transaction cannot.

---

## Verified defects

All five reported gaps were re-checked against current source. All five hold; two are mis-stated in
scope and one is understated. Citations are file + symbol, never line numbers.

### 1. Rows carry no space marker — HOLDS. This is the dangerous one.

`actor.set_instance_transforms` reads each row's pre-write pose with
`UInstancedStaticMeshComponent::GetInstanceTransform(Index, Request.Previous, bWorldSpace)`, where
`bWorldSpace` came from **the call's** `space` argument via `InstanceRpcParseSpace`. It echoes that
pose into `movedInstances[]` as `previousTransform`. The only record of which space it is in is the
top-level `space` field written by `InstanceRpcAddAxisEcho` — a sibling of the array, not part of it.

A batch written with `space: "local"` therefore produces local-space rows. `InstanceRpcParseSpace`
defaults to `world`. Replay those rows without repeating `space: "local"` and:

- the pre-flight accepts them (`index` is valid, `previousTransform` parses);
- `previousTransform` becomes the row's base, so every omitted field resolves;
- `UpdateInstanceTransform(Index, Target, /*bWorldSpace*/ true, ...)` writes component-local numbers
  as world coordinates;
- the post-write verification loop re-reads in the same wrong space, agrees, and increments
  `UpdatedCount`;
- the response is `updated: N`, with `mismatchedCount` absent.

The scatter is relocated to garbage, the package is dirtied, and nothing in the response says so.

**One mitigation the original report missed, stated here for honesty:** the destructive replay emits
*its own* `movedInstances[]`, carrying the true world-space pre-write poses. The scatter is
recoverable — by a caller who kept the response of a call they believed was a restore and did not
expect to need a second undo for. That is why this is High and not Critical.

### 2. `spatial.ground_instances` never echoes `space` — HOLDS, and is broader than reported.

`GroundRpcAddAxisEcho` writes `units` and `axis` only. It is shared by **all three** ground verbs
(`spatial.ground_actors`, `spatial.verify_grounding`, `spatial.ground_instances`), so none of them
echoes a space.

`spatial.ground_instances` declares no `space` parameter at all, and `GroundPlacement::SeatInstance`
(`Handlers/Spatial/GroundPlacementUtils.cpp`) hardcodes `/*bWorldSpace*/ true` on both its
`GetInstanceTransform` read and its `UpdateInstanceTransform` write. Its rows really are world-space,
and `actor.set_instance_transforms` really does default to world — so the naive replay is correct.
**Correct by coincidence:** the coincidence is that one verb's hardcode happens to equal another
verb's default, and no field, comment or doc in either file records the dependency. Change either
side and the pair silently stops agreeing.

### 3. "Survives an editor restart" is false — HOLDS.

The `InstancedMeshUtils.h` header comment claims the record, "unlike the transaction buffer[,]
survives an editor restart and can be applied selectively." Selective application is true. Surviving
a restart is not a property of anything the plugin does: no handler persists an RPC response, and
`system.job_status` results (`Handlers/System/JobControlHandler.cpp`) are in-memory. The record
survives exactly as long as the caller's own transcript — a property of the caller, not of the verb.

### 4. Inconsistent presence — HOLDS, wrong direction attributed.

`actor.set_instance_transforms` writes `movedInstances` unconditionally, possibly empty. **Both**
ground verbs gate on `MovedRows.Num() > 0` — `spatial.ground_instances` omits `movedInstances`, and
`spatial.ground_actors` omits `movedActors`, when nothing moved. So the inconsistency is
`actor.set_instance_transforms` against the entire ground family, not against `ground_instances`
alone. A caller must distinguish "absent because nothing moved" from "absent because this verb does
not produce one", and the response gives it no way to.

### 5. A `{index}`-only row is a no-op counted as `updated` — HOLDS, and is the only guard against a typo'd row key.

With no `previousTransform` and no pose fields, the row's base is `Request.Previous` — the instance's
current transform — and all three `ParseVectorFromJson` / `ParseRotatorFromJson` fallbacks resolve to
it. `Target == current`. `UpdateInstanceTransform` returns true, the readback agrees, `UpdatedCount`
increments. Accepted, counted, wrote nothing.

The round-trip fix means the *naive undo paste* no longer lands here — but this is also what a
**misspelled row key** degrades to. The dispatcher's `UNKNOWN_PARAMS` gate
(`Dispatch/RpcDispatcher.cpp`) is **top-level only**; it never inspects `instances[]` row keys. A row
of `{index: 7, previousTransfrom: {...}}` or `{index: 7, Location: {...}}` is accepted, writes
nothing, and is reported as updated. Refusing a pose-less row is the only structural check that
catches this.

---

## The actor-level siblings do NOT share the trap — and are safe for reasons that will not survive the next verb

`spatial.ground_actors` emits `movedActors[]` rows of `{actor, path, previousTransform}`, built in its
handler body from `FGroundSeatResult::PreviousTransform`. It is safe, structurally, on three counts:

- **No space ambiguity is expressible.** No `space` parameter exists anywhere on the actor path.
  `GroundPlacement::SeatActor` records `Actor->GetActorTransform()` and reverts with
  `Actor->SetActorTransform(...)`. World space is the only space there is.
- **No verb takes a row verbatim, so the silent replay cannot happen.** `actor.set_transform`
  (`Handlers/Actor/ActorTransformHandler.cpp`) takes one actor and top-level
  `location` / `rotation` / `scale`. A pasted `movedActors[]` row carries `path` and
  `previousTransform`, both unknown **top-level** keys — and `path` is not an actorName alias
  (`ActorNameParamUtils` declares `objectPath` and `actorPath`). The dispatcher refuses with
  `UNKNOWN_PARAMS`. Loud, not silent.
- The false-persistence claim (gap 3) is confined to `InstancedMeshUtils.h`; the shipped
  `Docs/wiki-src/spatial.md` text for `ground_actors` says only "That is the undo".

It **does** share gap 4 (`movedActors` omitted when empty).

**The ticket is ISM-scoped, but this is worth saying out loud:** the actor path is safe because nobody
has built the obvious batch restore verb yet. The moment an `actor.set_transforms` (plural) lands and
accepts a `movedActors[]` row shape, it inherits every one of these defects with actors instead of
instances. Whatever contract this ticket settles should be settled once, for both.

---

## Is a batched transaction affordable? Yes — the stated rationale is wrong

The `InstancedMeshUtils.h` header argues: `UpdateInstanceTransform` calls `Modify()` itself, `Modify()`
on an ISM serialises the entire `PerInstanceSMData` array, therefore wrapping a batch costs
O(instances written x total instances) — "hundreds of megabytes for a hundred-instance fix."

The multiplication does not happen. In `Editor/UnrealEd/Private/EditorTransaction.cpp`,
`FTransaction::SaveObject` creates an `FObjectRecord` only when the object has no record yet:

```cpp
FObjectRecords* ObjectRecords = &ObjectRecordsMap.FindOrAdd(UE::Transaction::FPersistentObjectRef(Object));
if (ObjectRecords->Records.Num() == 0)
{
    FObjectRecord* Record = new FObjectRecord(this, Object, nullptr, nullptr, 0, 0, 0, 0, 0, nullptr, nullptr, nullptr);
    ...
}
++ObjectRecords->SaveCount;
```

Every later `Modify()` on the same component (`UObject::Modify` -> `SaveToTransactionBuffer` ->
`GUndo->SaveObject`) only increments `SaveCount`. **One `FScopedTransaction` around the whole batch
costs exactly one full-object snapshot**, regardless of how many `UpdateInstanceTransform` calls it
wraps.

Magnitude: `FInstancedStaticMeshInstanceData` (`Components/InstancedStaticMeshComponent.h`) is a
single `FMatrix` — 128 bytes under LWC. A 10k-instance scatter is ~1.3 MB of `PerInstanceSMData`, plus
a HISM's `ClusterTree` / `SortedInstances` / `InstanceReorderTable`. Low single-digit MB, once — the
same price the editor already pays when a human drags one instance's gizmo in the details panel. The
"hundreds of megabytes" figure is the un-deduplicated number times a factor that does not exist.

Two further findings close the question:

- **The engine already supports the undo end-to-end.**
  `UInstancedStaticMeshComponent::PostEditUndo` (`Runtime/Engine/Private/InstancedStaticMesh.cpp`)
  does `InvalidateCachedBounds()`, `PrimitiveInstanceDataManager.Invalidate(PerInstanceSMData.Num())`,
  `FNavigationSystem::UpdateComponentData`, `MarkRenderStateDirty()`.
  `UHierarchicalInstancedStaticMeshComponent::PostEditUndo`
  (`Runtime/Engine/Private/HierarchicalInstancedStaticMesh.cpp`) adds
  `BuildTreeIfOutdated(/*Async*/ false, /*ForceUpdate*/ true)`. That is precisely the side-effect set
  `InstancedMeshUtils::FinishInstanceWrites` performs by hand after a write. Ctrl+Z reaches the same
  state the verb reaches — this is not a case where a transaction restores data the renderer never
  hears about.
- **`editor.undo` exists** (`Handlers/Editor/EditorCommandHandler.cpp`, `GEditor->UndoTransaction()`),
  so an agent caller can drive the undo it would gain.

**`BatchUpdateInstancesTransforms` is not the answer and is not needed for the transaction argument.**
`UInstancedStaticMeshComponent::BatchUpdateInstancesTransformsInternal` addresses a **contiguous** run
from `StartInstanceIndex` and walks `InstanceIndex++`; these verbs take arbitrary index sets. And
`UHierarchicalInstancedStaticMeshComponent::BatchUpdateInstancesTransformsInternal` simply loops
`UpdateInstanceTransform`, so on a HISM — the common scatter case — it calls `Modify()` N times
anyway. Its only win on a plain ISM is one navigation update instead of N. Irrelevant to undo cost.

**What the record still buys once the transaction exists** (so it should be kept, not deleted):
selective and partial restore; restore after intervening edits, which a stack-based undo cannot
reach; survival across an editor restart *in the caller's transcript*; and a machine-readable diff of
what a call did.

---

## Proposed contract: one self-describing `undo` envelope

Replace the bare `movedInstances[]` array with a single envelope carrying everything a replay needs,
so a caller who kept only the envelope has lost nothing:

```json
"undo": {
  "schema": 1,
  "verb": "actor.set_instance_transforms",
  "space": "local",
  "actorPath": "/Game/Maps/L_X.L_X:PersistentLevel.BP_Scatter_2",
  "component": "InstancedStaticMeshComponent0",
  "instanceCount": 10000,
  "instances": [ { "index": 7, "previousTransform": { "location": {}, "rotation": {}, "scale": {} } } ]
}
```

Invariants:

- **`space` sits beside the rows.** An envelope separated from its response still knows what its
  numbers mean. This is the whole fix for gaps 1 and 2.
- **`actorPath` + `component` + `instanceCount`** make the envelope name its own target and carry its
  own `expectedCount` guard, so a replay is a complete call rather than an array the caller must
  re-address by hand — which is exactly where the current trap lives.
- **`schema`** lets a future shape change be refused rather than misread.
- **Unconditional emission** by every producer, with an empty `instances` array when nothing moved.
  Absence then means "this verb produces no record", never "nothing moved". Closes gap 4.
- **`actor.set_instance_transforms` accepts the envelope** under a new `undo` parameter, as a complete
  call: with `undo` present, actor / component / space / instances / expectedCount are taken from it,
  and supplying a conflicting top-level one is a refusal, not a merge.
- **A row that requests no pose is refused** with a typed error naming the row index, instead of being
  counted as `updated`. Closes gap 5, and is the only guard against a misspelled row key since
  `UNKNOWN_PARAMS` never sees inside `instances[]`.

### Rejected alternatives

- **Per-row `space` marker.** Cheapest, and it closes gap 1. Loses because it repeats a per-call fact
  N times and closes *only* gap 1: the row still cannot name its component, cannot carry
  `expectedCount`, and a caller holding just the rows must still reconstruct the call by hand — the
  exact manual step the defect lives in.
- **Force rows unconditionally world-space regardless of the call's `space`.** Closes gap 1 with no
  schema change, and is genuinely tempting. Loses twice. It breaks the documented `space: "local"` use
  case — a caller reproducing a construction-script scatter gets an undo record in a space they must
  hand-compose the component transform to use, which is the precise hand-decomposition
  `actor.get_instances` was built to remove. And it makes the record's space silently differ from the
  call's space, replacing a discoverable inconsistency with an undiscoverable one.
- **A version/shape tag alone.** Necessary but insufficient: a tag says which shape you hold, not what
  the numbers mean. Kept as `schema` inside the envelope.
- **Refusing a pose-less row alone.** Closes gap 5 and nothing else. A local-space row replayed as
  world is *not* pose-less — it carries a complete `previousTransform` — so the refusal never fires on
  the dangerous path.
- **Document the hazard and change no code.** Rejected: the hazard is invisible at the point of use.
  The call succeeds, the readback agrees, `updated` counts every row. A doc note does not compete with
  a green result.
- **Delete the record once the transaction lands.** Rejected: undo is a stack that cannot reach past
  intervening edits, the transaction buffer does not survive an editor restart, and selective/partial
  restore is a real capability the record has and undo does not.

**Strongest argument against the envelope:** it is a breaking response-shape change on three verbs
that shipped yesterday, and it makes the simple case — "move ten instances down, paste the array
back" — carry five fields of ceremony it does not need. The counter is that the simple case is exactly
the one that silently destroys a scatter today, and that the shipped surface is one day old, which is
the cheapest this change will ever be.

---

**Fix:**

1. **Wrap each write batch in one `FScopedTransaction`** in `actor.set_instance_transforms` and in
   `spatial.ground_instances`' apply path. Open it after the all-or-nothing pre-flight passes and
   before the first `UpdateInstanceTransform`; close it before `FinishInstanceWrites`. Cost is one
   `FObjectRecord`, per the `FTransaction::SaveObject` dedup above. Do **not** reach for
   `BatchUpdateInstancesTransforms` — it needs contiguous indices and HISM's override loops
   `UpdateInstanceTransform` anyway.
2. **Rewrite the "NO FScopedTransaction, deliberately" paragraph in `InstancedMeshUtils.h`.** Its
   arithmetic is wrong and it is the reason the design exists. Replace it with the dedup finding and
   with the record's real remaining justification (selective restore, restore past intervening edits,
   machine-readable diff). Delete the "survives an editor restart" sentence outright — closes gap 3.
3. **Add `InstancedMeshUtils::WriteUndoEnvelope(Data, Verb, Actor, Component, bWorldSpace, Rows)`**
   beside `WriteComponentIdentity`, so all producers cannot disagree about the shape. Emit it
   unconditionally from `actor.set_instance_transforms` and `spatial.ground_instances`.
4. **Keep `movedInstances[]` for one release** as a deprecated top-level alias of `undo.instances`,
   documented as unsafe to replay on its own, so existing transcripts still parse.
5. **Accept `undo` as an input** on `actor.set_instance_transforms`: a new declared parameter, plus a
   refusal (`INVALID_ARGUMENT`) when a top-level `actorName` / `component` / `space` / `expectedCount`
   disagrees with the envelope's. Also refuse `schema` values this build does not know.
6. **Refuse a pose-less row.** After `previousTransform` resolution, if the row supplied no
   `previousTransform` and none of `location` / `rotation` / `scale`, send `INVALID_ARGUMENT` naming
   the row index and the accepted keys — before any write, consistent with the verb's existing
   all-or-nothing pre-flight. Closes gap 5.
7. **Add `space` to `GroundRpcAddAxisEcho`.** It serves `ground_actors`, `verify_grounding` and
   `ground_instances`; all three are world-space today, so the echo is a constant `"world"` that makes
   the contract explicit instead of coincidental. Closes gap 2.
8. **Tests** (extend `Tests/Spatial/TestGroundPlacement.cpp`, which already hosts the per-instance
   fixtures): (a) a `space:"local"` write's envelope replayed with no top-level `space` restores the
   original poses — the current-behaviour regression; (b) a row of `{index}` alone is refused, not
   counted; (c) `undo.instances` is present and empty when a `ground_instances` dry run moves nothing;
   (d) `editor.undo` after `actor.set_instance_transforms` restores the pre-batch transforms and
   leaves a HISM's cluster tree rebuilt.
9. **Document the contract where callers read it.** `Docs/wiki-src/spatial.md` has a `ground_actors`
   gotchas paragraph and **no** section for `spatial.ground_instances`; `Docs/wiki-src/actor.md` has
   nothing for `actor.get_instances` / `actor.set_instance_transforms`. The contract currently exists
   only inside handler summary strings.
10. **Note in `Docs/rpc-design.md`:** an echoed pre-state record is only an undo if a row is
    self-describing enough to replay without the response it arrived in — and it is never a substitute
    for a transaction that a dedup-per-object buffer makes affordable.

## History
- `#1-contract-audit` `OPEN` reporter — All five reported gaps verified in current source; all five hold. Gap 2 is broader than reported (`GroundRpcAddAxisEcho` serves all three ground verbs, none echoes a space). Gap 4's inconsistency is `set_instance_transforms` vs. the whole ground family — both `movedActors` and `movedInstances` are omitted when empty. Gap 5 is understated: the dispatcher's `UNKNOWN_PARAMS` gate is top-level only, so a misspelled `instances[]` row key degrades to exactly this silent no-op. Gap 1 confirmed dangerous and partially mitigated — the destructive replay emits its own correct world-space record, so the scatter is recoverable by a caller who kept it. The actor siblings do NOT share the trap: no `space` exists on the actor path, and a pasted `movedActors[]` row hits `UNKNOWN_PARAMS` on `path`/`previousTransform` because `path` is not an actorName alias. Decisive finding: `FTransaction::SaveObject` (`Editor/UnrealEd/Private/EditorTransaction.cpp`) creates an `FObjectRecord` only when the object has none, so one `FScopedTransaction` around a whole batch costs ONE snapshot (~1.3 MB on 10k instances), not O(written x total) — the recorded rationale for declining a transaction is arithmetically wrong, `PostEditUndo` on both ISM and HISM already performs the same refresh `FinishInstanceWrites` does, and `editor.undo` exists. Recommendation is to replace the design (add the transaction) and keep the record as a self-describing `undo` envelope, not to patch five holes.
