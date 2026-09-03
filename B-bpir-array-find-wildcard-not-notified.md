---
id: B-bpir-array-find-wildcard-not-notified
title: "`Array_Find`'s ItemToFind wildcard is never resolved, so every `call Array_Find(...)` fails to wire — the Array_Get fix in B-array-get-item-connection-fail did not generalise to its siblings"
status: OPEN
severity: Medium
category: bug
tags: [bpir, compile_bpir, array, wildcard, array_find, notify-pin-connection]
encounters: 1
lastSeen: 2026-09-02T20:05:00Z
---

# `Array_Find` cannot be wired: the wildcard `ItemToFind` pin stays untyped

## Repro

```
%pawn = call K2_GetPawn(Target: self)
%all  = call GetAllActorsOfClass(WorldContextObject: self, ActorClass: /Game/FPS/AI/BP_EnemyCharacter.BP_EnemyCharacter_C)
%idx  = call Array_Find(TargetArray: %all.OutActors, ItemToFind: %pawn)
```

```
[COMPILE_FAILED] Line 4: TryCreateConnection failed wiring data 'ReturnValue' -> 'ItemToFind'
```

`TargetArray` is wired first and is a concrete `TArray<AActor*>`; `%pawn` is an `APawn*`, which is a
plain upcast to the array's element type. Wiring it should be legal and is legal in the Blueprint
editor.

Inserting an explicit upcast does not help — the failure just renames itself:

```
%pa  = cast<Actor>(%pawn) [success -> @go]
@go:
%idx = call Array_Find(TargetArray: %all.OutActors, ItemToFind: %pa.AsActor)
```

```
[COMPILE_FAILED] Line 7: TryCreateConnection failed wiring data 'AsActor' -> 'ItemToFind'
```

The source pin is now literally `AActor*` and it still will not connect, which points at the
*destination* pin rather than the source: `ItemToFind` is still `PC_Wildcard` at wiring time.

## Why this looks like the Array_Get bug's sibling

`B-array-get-item-connection-fail` (DONE) was the same shape on `Array_Get`, and
[`bpir.instructions`](bpir.instructions.md) documents its fix:

> **End-to-end wiring:** `call Array_Get(...)` followed by `cast<T>(%h.Item)` now compiles cleanly.
> Previously wildcard propagation lagged behind downstream wiring, so `TryCreateConnection` failed
> with `wiring data 'Item' -> 'Object'`. The compiler now calls `NotifyPinConnectionListChanged`
> after wiring `TargetArray`, resolving the element type before later instructions use the output.

That fix was applied at the `Array_Get` call site rather than to the wildcard-array family, so every
other `UK2Node_CallArrayFunction` whose non-array pin is a wildcard is still broken. `Array_Find` is
the one I hit; `Array_AddUnique`, `Array_Contains`, `Array_RemoveItem` and `Array_Set` take the same
shape and are presumably affected identically (not tested here).

Note `Array_Add` **works** — `call Array_Add(TargetArray: $TraceIgnore, NewItem: %w2)` compiled fine
in the same session with `%w2` a `BP_AIWeaponStub_C` going into a `TArray<AActor*>`, so whatever
notification path `Array_Add` takes is the one the others need.

## Expected

`NotifyPinConnectionListChanged` after the `TargetArray` wire for every wildcard-array node, not for
`Array_Get` alone — or, better, a single rule in the wiring pass that re-notifies any node with a
remaining `PC_Wildcard` input after each successful connection.

**Workaround:** none found for `Array_Find` itself. I restructured the algorithm to avoid needing an
index at all (an authored `SquadSlot` int on the pawn replaced "find my position in the array"),
which changes the design rather than working around the defect.

severity rationale: impact=hard blocker with no workaround for this verb (the operation is
impossible; the caller must redesign around it) x reach=common (array index-of is ordinary Blueprint
work) -> Medium

## History
- `#1-filed` `OPEN` reporter — Hit on UE 5.8 / `EAContentExamples58` while authoring
  `AIC_Enemy::UpdateSquadRole`, which wanted "my index among all live `BP_EnemyCharacter`" to decide
  whether this enemy suppresses or flanks. Both spellings above were tried and both produced
  `TryCreateConnection failed wiring data ... -> 'ItemToFind'`; the second one proves the source pin
  is a concrete `AActor*`, so the wildcard on the destination is what fails. The contrasting
  `Array_Add` success in the same Blueprint session is the useful control: the family is not
  uniformly broken, so this is a missed call site rather than a missing capability. Related but
  distinct from `B-array-get-item-connection-fail` (DONE) — same root cause, different node, and
  that ticket's fix is described in the wiki as scoped to `Array_Get`'s `TargetArray` wiring.
