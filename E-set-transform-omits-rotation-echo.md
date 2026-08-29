---
id: E-set-transform-omits-rotation-echo
title: "`actor.set_transform` takes location, rotation and scale but echoes back only location and scale, and its own TRANSFORM_MISMATCH self-check ignores rotation too — so the one component with no readback is also the one component nothing verified"
status: OPEN
severity: Medium
category: ergonomic
tags: [actor, set_transform, rotation, echo, readback, response-shape, verification, transform-mismatch, every-session-verb]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The verb writes three components and reports two

`actor.set_transform` declares all three, `Handlers/Actor/ActorTransformHandler.cpp:33-35`:

```cpp
RPC_PARAM_OPT("location", "object", "New world location as {x,y,z} in unreal units (cm); omit to leave unchanged."),
RPC_PARAM_OPT("rotation", "object", "New world rotation as {pitch,yaw,roll} in degrees; omit to leave unchanged."),
RPC_PARAM_OPT("scale",    "object", "New 3D scale as {x,y,z} (1.0 = identity); omit to leave unchanged.")
```

It reads and applies all three (`:51-62`). The response carries two, `:76-77`:

```cpp
Data->SetArrayField(TEXT("location"), MakeVectorArray(NewLoc));
Data->SetArrayField(TEXT("scale"),    MakeVectorArray(NewScale));
```

There is no `rotation` field anywhere in the success payload.

## The self-check has the same hole, and that is the part worth fixing carefully

The handler re-reads all three components after the write — including rotation, `:68`:

```cpp
const FVector  NewLoc   = Found->GetActorLocation();
const FRotator NewRot   = Found->GetActorRotation();      // :68
const FVector  NewScale = Found->GetActorScale3D();

const bool bLocMatch   = NewLoc.Equals(Location, 1.0f);   // :71
const bool bScaleMatch = NewScale.Equals(Scale, 0.01f);   // :72
```

There is no `bRotMatch`. `NewRot` is read at `:68` and never used again — the whole file has exactly
one reference to it. The `TRANSFORM_MISMATCH` guard at `:79` therefore tests two of three
components:

```cpp
if (!bLocMatch || !bScaleMatch) {
    Ctx.SendError(TEXT("TRANSFORM_MISMATCH"), TEXT("Failed to set transform exactly"));
```

So rotation is the component that is neither echoed **nor** verified. A caller who sets a rotation
gets `ok`, no value back, and a success verdict that was computed without looking at it.

## The fix is one line, beside an existing correct example in the same file

`actor.get_transform`, 40 lines below, writes all three — `:117`, `:118-122`, `:123`:

```cpp
Data->SetArrayField(TEXT("location"), MakeVectorArray(Location));
TArray<TSharedPtr<FJsonValue>> RotArray;
RotArray.Add(MakeShared<FJsonValueNumber>(Rotation.Pitch));
RotArray.Add(MakeShared<FJsonValueNumber>(Rotation.Yaw));
RotArray.Add(MakeShared<FJsonValueNumber>(Rotation.Roll));
Data->SetArrayField(TEXT("rotation"), RotArray);
Data->SetArrayField(TEXT("scale"), MakeVectorArray(Scale));
```

Lifting `:118-122` into the `set_transform` block gives the two verbs the same response shape, which
they do not have today. Add `bRotMatch` alongside `:71-72` in the same change — `NewRot` is already
sitting there unused — using an angular tolerance rather than `FRotator::Equals`' component-wise
default, because the write goes through `SetActorRotation` and comes back off a quaternion, so an
exact component comparison will fail on values that are correct.

## Distinct from

- `E-foliage-get-instances-drops-scale` (IN-REVIEW, Low) — **literally this defect on another
  verb**: a response that omits one of the three transform components it holds. Whatever shape the
  fix takes there is the shape this wants. Not merged, because they are different verbs in different
  namespaces and each response is fixed in its own file; cross-linked so one fixer sees both.
- `E-actor-duplicate-no-rotation-scale` (OPEN, Low) — **do not merge this one.** That ticket is about
  `actor.duplicate` not accepting a rotation or scale on the way *in*; this is about
  `actor.set_transform` not reporting the rotation on the way *out*. Input axis versus echo axis, no
  shared code and no shared fix: adding parameters to `duplicate` leaves `set_transform`'s payload
  exactly as it is, and adding a field to `set_transform`'s payload gives `duplicate` no new
  parameter.
- `B-sequencer-get-binding-transform-rotation-fields-rotated` (DONE, High) — a rotation triple
  returned in the *wrong slots*. That is silent wrong data and correctly rated High. Here nothing
  wrong is returned; the field is simply absent, which a caller can detect.

## Dedup

Board-wide search across all statuses for `set_transform` echo/rotation tickets: the only rotation-
adjacent files are `E-actor-duplicate-no-rotation-scale` and
`B-sequencer-get-binding-transform-rotation-fields-rotated`, both distinguished above. Nothing owns
`actor.set_transform`'s response shape.

## History
- `#1-two-of-three-echoed` `OPEN` reporter — Measured and re-derived at HEAD. `actor.set_transform` declares `location`, `rotation` and `scale` (`Handlers/Actor/ActorTransformHandler.cpp:33-35`), applies all three (`:51-62`), and echoes only two — `SetArrayField("location")` at `:76` and `SetArrayField("scale")` at `:77`, with no `rotation` field in the payload. The sibling `actor.get_transform` in the same file writes all three (`:117`, `:118-122`, `:123`), so the corrected form is five lines below the defect. Second finding, from re-deriving rather than from the report: the handler's own post-write verification skips rotation too — `NewRot` is read at `:68` and referenced nowhere else in the file, and the `TRANSFORM_MISMATCH` guard at `:79` tests only `bLocMatch` (`:71`) and `bScaleMatch` (`:72`). Rotation is therefore the one component that is neither echoed nor checked, and the success verdict is computed without consulting it. Ask: echo `rotation` from the post-write `NewRot` using `get_transform`'s pitch/yaw/roll array shape, and add a `bRotMatch` with an angular tolerance (not component-wise `FRotator::Equals` — the value round-trips through a quaternion in `SetActorRotation`, so exact component equality would fail on correct writes). Cross-linked `E-foliage-get-instances-drops-scale` (IN-REVIEW, Low) as the same defect on another verb, and `E-actor-duplicate-no-rotation-scale` (OPEN, Low) with an explicit do-not-merge: that one is the input axis (`duplicate` accepts no rotation/scale), this is the echo axis, and neither fix touches the other's file. Severity Medium, argued rather than inherited from the Low precedent: impact class is Low on its own reading — a response spill that costs one extra `actor.get_transform` — but the rubric's reach modifier bumps a Low-impact gap up one level on a method that runs in almost every session, and `actor.set_transform` is a core placement verb hit constantly, where `foliage.get_instances` is not. Explicitly declining a second bump to High for the unverified rotation: the rotation really is applied, so no wrong or stale value is returned, and High's band is silent false-success on data the caller trusts.
