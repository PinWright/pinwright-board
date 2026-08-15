---
id: B-set-camera-fov-cine-recalc-overwrites
title: "misc.set_camera_fov on a CineCamera writes a FieldOfView that RecalcDerivedData overwrites from the clamped focal length on PostLoad"
status: OPEN
severity: Medium
category: bug
tags: [camera, cine-camera, derived-state, persistence, silent-revert, misleading-success]
---

# `misc.set_camera_fov` writes a derived field on a CineCamera

Same shape as `B-spline-point-scale-water-derived`.

`Handlers/Utility/MiscHandler.cpp:261` calls `CamComp->SetFieldOfView(...)` (verb registered
`:220`); `misc.create_camera` does the same at `:203` (verb `:155`, and its own doc string
advertises `cine=true` / `cameraClass="cine"`). The actor scan at `:243` is
`TActorIterator<ACameraActor>` and `ACineCameraActor` derives from `ACameraActor`, so cine
cameras are always in scope. The response at `:265` echoes the **requested** fov, not the
component's, so even an in-editor clamp goes unnoticed.

## Root cause (source-verified against `C:\UE_5.8`)

`UCineCameraComponent::SetFieldOfView`
(`Runtime/CinematicCamera/Private/CineCameraComponent.cpp:258-273`) calls
`Super::SetFieldOfView` — inline `{ FieldOfView = InFieldOfView; }`,
`Runtime/Engine/Classes/Camera/CameraComponent.h:46` — and back-solves `CurrentFocalLength`
**without clamping and without calling `RecalcDerivedData()`**. So the value reads back exactly
and serializes that way.

`UCineCameraComponent::PostLoad` (`CineCameraComponent.cpp:137-148`) calls `RecalcDerivedData()`
at `:146`. `RecalcDerivedData` (`:522-541`) clamps `CurrentFocalLength` to
`LensSettings.Min/MaxFocalLength` and then **overwrites**
`FieldOfView = GetHorizontalFieldOfViewInternal(false)`. It also runs from
`PostInitProperties` (`:250-255`) and `PostEditChangeProperty` (`:220`).

The lens range is narrow in practice: `FCameraLensSettings` defaults to
`MinFocalLength(50.f), MaxFocalLength(50.f)`
(`Runtime/CinematicCamera/Public/CineCameraSettings.h:94-101`) and `BaseEngine.ini:3622-3627`
ships fixed primes. On any prime-lens cine camera **every** `set_camera_fov` value reverts on
reload to the FOV implied by that prime.

The engine already treats this setter as wrong for the class:
`UCLASS(HideFunctions = (SetFieldOfView, SetAspectRatio), ...)` (`CineCameraComponent.h:20`).

## Authoritative state

`UCineCameraComponent::SetCurrentFocalLength` (`CineCameraComponent.cpp:275-279`, which does
call `RecalcDerivedData`) plus `Filmback`. If a FOV is genuinely wanted, widen
`LensSettings.MinFocalLength/MaxFocalLength` first and report the post-`RecalcDerivedData` FOV
rather than the requested one.

## Fix shape

Per `Docs/rpc-design.md` §5a: on a cine camera either forward to the focal-length setter (the
mapping is exact given a filmback) or refuse with `DERIVED_PROPERTY` naming it. Either way,
report the FOV read back off the component instead of echoing the request.

## History
- `#1-found-by-sweep` `OPEN` reporter — Found by the derived-state sweep run alongside `B-spline-point-scale-water-derived`. Not reproduced live; the mechanism is read out of UE 5.8 source at the lines quoted above. The trigger threshold depends on the mounted lens preset — on the "Universal Zoom" default (4–1000mm, `BaseEngine.ini:3630-3631`) only extreme values clamp, on a fixed prime every value reverts.
