---
id: B-bpir-array-find-wildcard-not-notified
title: "`Array_Find`'s ItemToFind wildcard is never resolved, so every `call Array_Find(...)` fails to wire — the Array_Get fix in B-array-get-item-connection-fail did not generalise to its siblings"
status: DONE
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
- `#2-mismatch-not-wildcard` `IN-REVIEW` developer — Verified against plugin + engine source, no editor run. **The filed mechanism is false: `ItemToFind` is not left wildcard.** `FCodeNodeEmitter::CreateCallFunctionNode` (`Compiler/CodeNodeEmitter.cpp`) already emits *every* `ArrayParm` function — not just `Array_Get` — as `UK2Node_CallArrayFunction`, whose `NotifyPinConnectionListChanged` (`K2Node_CallArrayFunction.cpp:84-120`) gives `ArrayTypeDependentParams` (`ItemToFind`) the array's element type when `TargetArray` is wired; and a wildcard input would *accept* an `AActor*`, so a still-wildcard pin cannot produce this failure. What the repro hit is a real downcast: `GetAllActorsOfClass` carries `DeterminesOutputType="ActorClass"` (`GameplayStatics.h:97`), and the class literal reaches `UK2Node_CallFunction::PinDefaultValueChanged` -> `FDynamicOutputHelper::ConformOutputType` through `TrySetDefaultValue`, so `OutActors` is `TArray<BP_EnemyCharacter_C*>`, `ItemToFind` is `BP_EnemyCharacter_C*`, and both `APawn*` and `AActor*` are base classes of it — the editor refuses the same wire. The real defect is the error text, which named pins but no types and so read like an unresolved wildcard. Fixed: the data-wiring failure in `FBpirCompiler::WireDataPins` now reads `TryCreateConnection failed wiring data 'X' (<source type>) -> 'Y' (<target type>): <CanCreateConnection reason>`. The wiki's `Array_Get` note (which described a per-call-site `NotifyPinConnectionListChanged` call that does not exist) is rewritten in `bpir.instructions.md` to state the real mechanism, the subclass rule, and the `GetAllActorsOfClass` typing. Files: `Source/PinWright/Private/Compiler/BpirCompiler.cpp` (error message), `docs/wiki-src/bpir.instructions.md`, `CHANGELOG.md`, new `Source/PinWright/Private/Tests/Bpir/TestBpirWildcardResolution.cpp`. Tests: `PinWright.bpir.compiler.array_wildcard.SubclassItemWires` (Array_Find/Contains/AddUnique/RemoveItem with a Pawn into `TArray<AActor*>` wire and their item pin is typed Actor — the stale-ticket evidence) and `PinWright.bpir.compiler.array_wildcard.MismatchErrorNamesPinTypes` (Pawn into `TArray<ACharacter*>` fails and the error names `Pawn` and `Character`; fails on the old pin-names-only message). Compile-checked with -SingleFile; not run.
- `#3-review-reporter-shape-test` `IN-REVIEW` developer — Review follow-up: added the reporter's actual shape as `PinWright.bpir.compiler.array_wildcard.DynamicOutputArrayMismatchNamesTypes` — a successful compile of `GetAllActorsOfClass(WorldContextObject: self, ActorClass: ACharacter)` asserts `OutActors` is an array of `ACharacter` (DeterminesOutputType), then `Array_Find(TargetArray: %all.OutActors, ItemToFind: $Probe)` with an `AActor*` probe fails and the error names `Actor` and `Character`. `SubclassItemWires` stays as the routing guard. Test helpers in `TestBpirWildcardResolution.cpp` are now called namespace-qualified.
- `#4-review-fixes` `IN-REVIEW` developer — Re-review NIT: `MismatchErrorNamesPinTypes` and `DynamicOutputArrayMismatchNamesTypes` (`Tests/Bpir/TestBpirWildcardResolution.cpp`) now assert on the one `TryCreateConnection failed wiring data ... 'ItemToFind'` error and require `(Pawn|Actor Object Reference)` and `(Character Object Reference)` in it, instead of loose any-error `Actor`/`Character` substrings. fastcheck OK; not run in an editor.
- `#5-run1-fixes` `IN-REVIEW` developer — Run1: `MismatchErrorNamesPinTypes` and `DynamicOutputArrayMismatchNamesTypes` failed only on the test's target-type expectation. `ItemToFind` is a const-ref param, so `UEdGraphSchema_K2::TypeToText` renders it `Character Object Reference (by ref)` (actual: `'Probe' (Pawn Object Reference) -> 'ItemToFind' (Character Object Reference (by ref)): ...`). The asserts now anchor on the pin name (`'Probe' (X Object Reference)`, `'ItemToFind' (Character Object Reference`), and the wiki example in `bpir.instructions.md` quotes the real message. Compiler unchanged. The `Unknown value reference format: 'ACharacter'` warning is unrelated log noise, filed as B-bpir-bare-class-arg-logs-unknown-value-warning.
- `#6-verified-linux` `DONE` tester — Run3 on PinWright 7230b41d, UE 5.8 Linux, all non-skipped in run3/full. `PinWright.bpir.compiler.array_wildcard.SubclassItemWires`: Array_Find/Contains/AddUnique/RemoveItem with a Pawn into `TArray<AActor*>` wire, and the item pin is typed Actor. The wildcard family resolves, so the filed mechanism (ItemToFind left wildcard) does not exist on this tree. `PinWright.bpir.compiler.array_wildcard.DynamicOutputArrayMismatchNamesTypes` shows the reporter's real shape: `GetAllActorsOfClass(ActorClass: X)` types OutActors as an array of X (DeterminesOutputType), so a base-class item is a genuine downcast that the editor also refuses. The error now names both pin types and the schema reason. `PinWright.bpir.compiler.array_wildcard.MismatchErrorNamesPinTypes` passed too. Doc: `bpir.instructions.md` states the subclass rule, the GetAllActorsOfClass typing and the real message. The reporter's 'none found' workaround has an answer: pass `ActorClass: Actor`, or an item of the array's element type. The asked-for re-notify change was not made because the tests show it is not needed.
