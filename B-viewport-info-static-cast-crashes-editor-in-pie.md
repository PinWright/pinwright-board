---
id: B-viewport-info-static-cast-crashes-editor-in-pie
title: "`system.inspect.get_viewport_info` hard-crashes the editor during PIE — an unchecked `static_cast<FEditorViewportClient*>` onto the sibling `UGameViewportClient` that PIE swaps into the active viewport, then reads camera members past the end of the allocation"
status: OPEN
severity: Critical
category: bug
tags: [system, inspect, get_viewport_info, environment, crash, access-violation, static-cast, pie, viewport-client, editor-kill]
encounters: 1
costly: 1
lastSeen: 2026-09-10T00:00:00Z
---

# A read-only verb kills the editor whenever PIE is running

Observed 2026-09-10 driving a PIE session through the MCP against UE 5.8, host project
`X:\src\unreal\unreal-fpv-new`, plugin at `d5dfb11b`. PIE was running **inside the
level-editor viewport** (no separate standalone PIE window). One
`system.inspect.get_viewport_info` call took the editor down with

```
EXCEPTION_ACCESS_VIOLATION
UnrealEditor-PinWright.dll!AutoHandler_359_()
  Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp:1006
  via FJsonObjectSharedStringStorage::SetObjectField
```

## Cause

`Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp:1003-1004`

```cpp
if (FEditorViewportClient* ViewportClient =
        static_cast<FEditorViewportClient*>(Viewport->GetClient()))
```

`FViewport::GetClient()` returns `FViewportClient*`. `FEditorViewportClient` and
`UGameViewportClient` are **siblings**, not related by inheritance — they meet only at
`FCommonViewportClient`:

- `FEditorViewportClient : FCommonViewportClient, FViewElementDrawer, FGCObject` — `C:\UE_5.8\Engine\Source\Editor\UnrealEd\Public\EditorViewportClient.h:345`
- `UGameViewportClient : UScriptViewportClient` -> `UScriptViewportClient : UObject, FGameplayViewportClient` (`ScriptViewportClient.h:19`) -> `FGameplayViewportClient : FCommonViewportClient` (`GameplayViewportClient.h:8`)

So the cast is a downcast onto the wrong branch, and it **cannot return null** —
`GetClient()` is non-null during PIE — which makes the `if` guard dead code. The body
then runs on a pointer that is not the type it claims.

**Why PIE-in-viewport specifically.** `UEditorEngine::GetActiveViewport()`
(`C:\UE_5.8\Engine\Source\Editor\UnrealEd\Private\PlayLevel.cpp:1958-1971`) returns
`SLevelViewport::GetActiveViewport()` (`SLevelViewport.cpp:2985-2988`), and
`SLevelViewport::StartPlayInEditorSession` swaps that pointer to a scene viewport built
on the **game** client:

```cpp
ActiveViewport = FSceneViewport::Create(PlayClient.Get(), ViewportWidget.ToSharedRef());
```

— `C:\UE_5.8\Engine\Source\Editor\LevelEditor\Private\SLevelViewport.cpp:4991`
(`PlayClient` is a `UGameViewportClient*`, signature at `:4941`).

Outside PIE the active viewport's client really is the `FLevelEditorViewportClient` and
the cast is accidentally correct, which is why the verb works normally and only ever
crashes under PIE.

**The faulting dereference** is `EnvironmentHandler.cpp:1006-1007`:

```cpp
Resp->SetObjectField(TEXT("cameraLocation"),
    JsonBuilders::BuildVectorJson(ViewportClient->GetViewLocation()));
```

`FEditorViewportClient::GetViewLocation()` (`EditorViewportClient.h:561-566`) is a
non-virtual inline reading `ViewTransformPerspective` / `ViewTransformOrthographic`
(`EditorViewportClient.h:1965,1968`) — offsets deep inside a class far larger than
`UGameViewportClient`, so the load walks past the end of the allocation. It returns
`const FVector&`, and the X/Y/Z load happens inside `JsonBuilders::BuildVectorJson`,
which is `inline` (`Source/PinWright/Private/Utils/JsonBuilders.h:41-48`) and folds into
the handler — which is why the reporter attributes the frame to `AutoHandler_359_` at
`:1006` with the JSON symbol on the stack.

Ruled out: `Resp` is validly constructed at `:996`; `BuildVectorJson` always returns a
valid `TSharedPtr`; the null-`GEditor` / null-viewport path is guarded at `:997`; the
`else` branch reporting "Viewport info not available in this context" (`:1017`) is
unreachable under PIE because the active viewport is non-null. Lines `:1008-1009`
(`GetViewRotation`) and `:1010` (`ViewFOV`, a direct member read) fault identically if
`:1006` alone is fixed.

## The same latent bug at three sibling call sites

All reachable during PIE-in-viewport, all `static_cast` on `GetClient()` with an equally
dead null guard:

- `Source/PinWright/Private/Handlers/Editor/EditorHandlerUtils.h:38` — `ResolveActiveLevelViewportClient()`, the **shared helper**; `if (!Client) return nullptr;` at `:39-42` is dead and `Client->GetWorld()` at `:44` is a virtual call on the bogus pointer.
- `Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp:82-84` — `ForceRedrawActiveViewport()` calls `ViewportClient->Invalidate(true, true)`.
- `Source/PinWright/Private/Handlers/Actor/ActorNudgeHandler.cpp:86`.

## Provenance

Only one commit ever touched this hunk: `8962f163` (the mechanical `EditorAutomation` ->
`PinWright` rename); `git log -S"static_cast<FEditorViewportClient*>"` on this file
returns that same single commit. The camera fields themselves were added under
`E-viewport-info-camera-transform` (DONE), whose `#3-corrective-fix` history entry
**prescribes the unchecked cast** and records that it was verified only in a non-PIE
editor. That ticket's fix is what introduced this crash.

## Fix

A checked discriminator, not a `static_cast`. Options, in order of directness:

1. Bail (or take a game-viewport branch) when `GEditor->PlayWorld` is set or `GEditor->GetPIEViewport() == Viewport`.
2. Route through `FLevelEditorModule::GetFirstActiveLevelViewport()->GetLevelViewportClient()`, which is typed. `SLevelViewport` keeps the editor client alive in `InactiveViewport` / `LevelViewportClient` during PIE, so the editor camera stays readable rather than the verb losing the fields.

Fix `EditorHandlerUtils.h:38` in the same pass — it is the shared helper and the other two
sites repeat the pattern. A live regression test needs a PIE session, so the cheap durable
half is a source contract asserting no `static_cast<FEditorViewportClient*>` on a
`GetClient()` result anywhere in `Source/`.

## Workaround

Do not call `system.inspect.get_viewport_info` (or `editor.*` verbs routed through
`ResolveActiveLevelViewportClient`) while PIE is running. There is no in-band signal that
the call is unsafe — the verb is read-only and advertises nothing about PIE.

## Related

- `E-viewport-info-camera-transform` (DONE, Low) — added the camera fields and prescribed the crashing cast.
- `B-editor-bookmark-roundtrip-noop` (IN-REVIEW, High) and `B-misc-create-bookmark-silent-noop` (IN-REVIEW, High) — both mention the same `GetClient()` pattern; worth checking against this cause.
- `B-editor-quit-crash-pie-active` (IN-REVIEW, Critical) and `B-editor-screenshot-pie-gpu-pagefault-during-shader-late-association` (IN-REVIEW, Critical) — other PIE-active editor kills, distinct root causes.

## History
- `#1-access-violation-on-pie-in-viewport` `OPEN` reporter — Filed from a PIE session on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `d5dfb11b`, PIE running inside the level-editor viewport with no standalone window. One `system.inspect.get_viewport_info` call produced `EXCEPTION_ACCESS_VIOLATION` in `AutoHandler_359_` at `EnvironmentHandler.cpp:1006`. Root-caused in source: `static_cast<FEditorViewportClient*>(Viewport->GetClient())` at `:1003-1004` is a cross-branch downcast (the two client classes meet only at `FCommonViewportClient`) whose null guard can never fire, and `SLevelViewport::StartPlayInEditorSession` (`SLevelViewport.cpp:4991`) swaps a `UGameViewportClient` into the very pointer `GEditor->GetActiveViewport()` returns — so the verb is safe outside PIE and fatal inside it. Confirmed unfixed at HEAD (`d5dfb11b`, clean tree, HEAD blob byte-identical to the working copy over `:992-1021`); the only commit touching the hunk is the `8962f163` rename. Found the same unchecked cast at `EditorHandlerUtils.h:38` (the shared helper), `ViewportHandler.cpp:82-84` and `ActorNudgeHandler.cpp:86`. Severity Critical per the rubric's editor-crash class. Marked `costly`: the crash ended the PIE session and the editor, losing the run. No fix attempted.
