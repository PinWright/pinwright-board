---
id: E-create-camera-cine-doc
title: "misc.create_camera doc advertises '(or CineCameraActor)' but offers no way to spawn one"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [misc, create_camera, camera, docs, cinematic]
---

# misc.create_camera doc advertises "(or CineCameraActor)" but offers no way to spawn one

The `misc.create_camera` wiki summary (and the `misc` namespace index line) reads:

> Spawn an ACameraActor **(or CineCameraActor)** in the active level. Use misc.set_camera_fov afterward to configure FOV.

The parenthetical "(or CineCameraActor)" implies the caller can choose to spawn a
cinematic camera. But the handler always spawns a plain `ACameraActor`, and there
is **no parameter** to select the cine variant — the param allowlist is strictly
`[cameraName, location, rotation, fov]`. Every plausible cine selector is rejected.

This bites the common "cine-style / hero-shot camera" task: the agent asks for a
cinematic camera, `create_camera` silently hands back a plain CameraActor (so the
exposure/filmback/focus knobs of a CineCameraActor are absent), and the doc's own
wording suggested that should have been selectable. The capability is reachable —
just through a *different namespace* (`actor.spawn` with `classPath:"CineCameraActor"`,
verified below) — so this is not a missing feature, it's misleading doc text that
sends callers down the wrong method.

**What it should do:** either (a) add an opt-in param to `misc.create_camera`
(e.g. `cine:true` / `cameraClass:"CineCameraActor"`) so the documented "(or
CineCameraActor)" is actually reachable, or (b) drop the "(or CineCameraActor)"
parenthetical from the summary and point callers to
`actor.spawn{classPath:"CineCameraActor"}` for the cine variant.

**Workaround:** spawn the cine camera via the sibling RPC
`actor.spawn{classPath:"CineCameraActor"}`, then `misc.set_camera_fov` it.

## Repro (verbatim)

Doc claim — `call("misc.create_camera")` wiki page / `misc` index, line 21:
`Spawn an ACameraActor (or CineCameraActor) in the active level.`

Actual spawn class is always plain CameraActor:
`call("misc.create_camera", {cameraName:"OracleProbeCamPlain", location:{x:600,y:-400,z:250}, rotation:{pitch:-15,yaw:135,roll:0}, fov:50})`
->
`{"cameraName":"OracleProbeCamPlain", ... "actorClass":"CameraActor"}`

No way to request the cine variant — every selector is rejected:
`call("misc.create_camera", {cameraName:"X", cine:true})`
-> `[UNKNOWN_PARAMS] Unknown parameter(s) for 'misc.create_camera': [cine]. Valid parameters: [cameraName, location, rotation, fov]. ...`
`call("misc.create_camera", {cameraName:"X", cameraType:"cine"})`
-> `[UNKNOWN_PARAMS] ... [cameraType]. Valid parameters: [cameraName, location, rotation, fov]. ...`
`call("misc.create_camera", {cameraName:"X", cineCamera:true})`
-> `[UNKNOWN_PARAMS] ... [cineCamera]. Valid parameters: [cameraName, location, rotation, fov]. ...`

Cine variant IS reachable via the sibling RPC (so this is doc/ergo, not a gap):
`call("actor.spawn", {classPath:"CineCameraActor", actorName:"OracleProbeCineViaSpawn"})`
-> `{... "classPath":"/Script/CinematicCamera.CineCameraActor", ... "actorClass":"CineCameraActor"}`

## History
- `#1-initial-repro` `OPEN` reporter — `misc.create_camera` summary says "Spawn an ACameraActor (or CineCameraActor)" but always returns `actorClass:"CameraActor"` and rejects every cine-selecting param (`cine`/`cameraType`/`cineCamera`) with `[UNKNOWN_PARAMS]` against the fixed allowlist `[cameraName, location, rotation, fov]`. Cine variant only reachable via sibling `actor.spawn{classPath:"CineCameraActor"}` (verified spawns `actorClass:"CineCameraActor"`). Fix: add a cine opt-in param, or drop the misleading parenthetical and point to actor.spawn.
- `#2-additional-hero-shot-task` `OPEN` reporter — Additional evidence (independent cinematic establishing-shot task, realism mode): still reproduces unchanged. Wiki page `misc.create_camera.md:7` still reads `Spawn an ACameraActor (or CineCameraActor) in the active level.`, and the handler source confirms the `(or CineCameraActor)` is unreachable — `MiscHandler.cpp:175` unconditionally does `World->SpawnActor<ACameraActor>(...)` with no class-selection branch, and the registered param allowlist (`MiscHandler.cpp:153-157`) is exactly `[cameraName, location, rotation, fov]`. The hero-shot agent's `misc.create_camera{cine:true}` was rejected `[UNKNOWN_PARAMS] ... [cine]. Valid parameters: [cameraName, location, rotation, fov].`, forcing the documented `actor.spawn{classPath:"CineCameraActor"}` workaround.
- `#3-add-cine-opt-in` `IN-REVIEW` developer — Implemented fix option (a): the documented "(or CineCameraActor)" variant is now reachable on `misc.create_camera` itself. Added two opt-in params to the registration — `cine` (boolean) and `cameraClass` (string, accepts "cine"/"cinecamera"/"cinecameraactor", camelCase alias) — and a class-selection branch that spawns `ACineCameraActor` (which derives from `ACameraActor`, so the shared label/FOV/response code is untouched) when opted in, else the historical plain `ACameraActor` (default behavior unchanged). The response now also echoes `cine:<bool>`; `actorClass` already reflects the real class via `AddActorVerification` (`AssetUtils.cpp:942`), so a cine spawn reports `actorClass:"CineCameraActor"`. Also reworded the summary string so the parenthetical no longer over-promises: now `"Spawn an ACameraActor (or, with cine=true / cameraClass=\"cine\", an ACineCameraActor)..."`. The misleading text originated solely in the registered summary at `MiscHandler.cpp:152` (the `Docs/wiki-src/misc.md` overlay has no `create_camera` section), so the summary edit fixes the generated wiki page + namespace index. `CinematicCamera` is already a `PublicDependencyModuleNames` entry (Build.cs:19), so `#include "CineCameraActor.h"` links cleanly. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Utility/MiscHandler.cpp`. Test: added `FMiscCreateCameraCineVariantTest` (`EditorAutomationRpcGateway.misc.create_camera.CineVariantSpawnsCineCameraActor`) in `Source/EditorAutomationRpcGateway/Private/Tests/Utility/TestUtilityHandlers.cpp` — invokes the production handler via `InvokeHandlerWithCapture` and asserts default spawns `CameraActor`, while `cine:true` and `cameraClass:"cine"` both spawn `CineCameraActor` (would fail if the class-selection branch were reverted, since the old handler unconditionally spawned plain CameraActor and rejected the cine param).
