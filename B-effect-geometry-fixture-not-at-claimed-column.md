---
id: B-effect-geometry-fixture-not-at-claimed-column
title: "TestEffectGeometrySurfaceFilter's fixture registers at world origin, not the isolated column its comment claims"
status: OPEN
severity: Low
category: bug
tags: [tests, spatial, fixtures, latent-trap]
encounters: 1
lastSeen: 2026-08-16T00:00:00Z
---

# A fixture comment promises column isolation the code does not provide

`Source/PinWright/Private/Tests/Spatial/TestEffectGeometrySurfaceFilter.cpp:42-46` documents a
dedicated world column at `(-880000, 640000)`, chosen to keep the suite clear of
`TestPlacementHandlers.cpp` (`223450, 167890`) and `TestGroundPlacement.cpp` (`331100, 274500`).
The fixture does not land there.

`:190-191` spawns a bare `AActor` with a spawn transform at `(-880000, 640000, 500)`, but a
root-less `AActor` discards its spawn transform — `AActor::PostSpawnInitialize` applies it only
when a root component already exists. `SetRootComponent` at `:209` attaches the billboard
afterwards and does not move it: its relative transform is identity with no parent, so the
component registers at world **(0, 0, 0)**.

## Why it is only Low today, and why it should still be fixed

Harmless as written: all three tests in the file are pure predicate checks on
`SpatialTraceUtils::IsEffectGeometryActor` and `FSpatialHitFilter::Matches`. None performs a
trace, so nothing reads the location, and `ON_SCOPE_EXIT` destroys the actor inside the same
test while automation runs serially. All three pass (integration pass 10, 3758-test run).

It is a **latent trap**, not a cosmetic one. The file's own comment invites the next author to add
a trace-based case to this suite, and that case would silently run at world origin — the single
most crowded column in any level — while the comment overhead says it is 9 km clear of everything.
That is the failure mode the sibling suites' column convention exists to prevent.

## Fix

Either set the root before the spawn transform can be applied, or move the actor explicitly after
`SetRootComponent`:

```cpp
Card->SetRootComponent(Billboard);
Card->SetActorLocation(FVector(-880000.0, 640000.0, 500.0));
Billboard->RegisterComponent();
```

Verify by reading `Billboard->GetComponentLocation()` back rather than trusting the spawn call —
the same readback discipline `rpc-design.md` §4 requires of the verbs themselves.

## Also worth aligning while in the file

Every other fixture in this repo calls `Actor->AddInstanceComponent(Comp)` before
`SetRootComponent` / `RegisterComponent` (`TestGetComponentsLargePayload.cpp:135-137`,
`TestEnvironmentHandlers.cpp:56-58`, `TestActorDuplicateComponentHandler.cpp:37-39`,
`TestActorDescribeBuilder.cpp:95-96`, `TestObjectCallFunctionHandler.cpp:122-124`). This one does
not. It is genuinely not required — `AddInstanceComponent` populates `InstanceComponents`, not the
`OwnedComponents` array `GetComponents` walks — so this is house style, not correctness.

The `SetCollisionEnabled` **after** `RegisterComponent` ordering in this fixture is correct and
should not be "fixed": `UBillboardComponent`'s constructor sets the `NoCollision` profile
(`BillboardComponent.cpp:279`), which resolves during registration, so the explicit
`SetCollisionEnabled(QueryOnly)` has to come afterwards to survive. `FBodyInstance::SetCollisionEnabled`
calls `InvalidateCollisionProfileName()` before assigning, so nothing re-applies `NoCollision`.

## History

- `#1-found-during-integration-pass-10` **OPEN** — Reporter. Found while auditing the new tests
  ahead of the build in integration pass 10, as part of checking the change author's own
  "could not confirm without a compiler" list. Not a build or test failure; the suite is green.
  Filed rather than fixed inline because the fix touches a test file in a changeset that had
  already built and passed, and re-running the 3758-test suite to land a comment-vs-code
  alignment was not a trade worth making mid-pass.
