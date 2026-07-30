---
id: B-editor-quit-crash-pie-active
title: "editor.quit with a live PIE session crashes the editor on shutdown (EXCEPTION_ACCESS_VIOLATION 0x60 in UAssetEditorSubsystem::FindEditorsForAssetAndSubObjects) — quit must end PIE first"
status: IN-REVIEW
severity: Critical
category: bug
tags: [editor-quit, shutdown, crash, access-violation, pie, teardown-order, asset-editor-subsystem, silent-failure]
encounters: 2
lastSeen: 2026-07-30T15:26:34+03:00
---

# `editor.quit` with PIE active crashes the editor during shutdown

Calling `editor.quit` while a Play-In-Editor session is running kills the editor
process with an unhandled `EXCEPTION_ACCESS_VIOLATION reading address
0x0000000000000060` during engine shutdown. Hit twice in one session on
2026-07-30, UE 5.8, PDS project.

The RPC itself reports success — the quit returns the normal
`{"requested":true,"dirtyCount":0,...}` ack and only *then* does the process die
in teardown. Nothing in the MCP response, and nothing on the calling agent's
side, indicates that the editor did not exit cleanly.

## Crash anatomy (captured callstack, engine frames only)

```
UAssetEditorSubsystem::FindEditorsForAssetAndSubObjects()  AssetEditorSubsystem.cpp:295
UAssetEditorSubsystem::CloseAllEditorsForAsset()           AssetEditorSubsystem.cpp:309
UEditorEngine::EndPlayMap()                                PlayLevel.cpp:437
FAssetEditorViewportLayout::~FAssetEditorViewportLayout()  AssetEditorViewportLayout.cpp:137
  ... Slate widget/window destructor chain ...
FSlateApplication::CloseAllWindowsImmediately()            SlateApplication.cpp:3242
FSlateApplication::Shutdown()                              SlateApplication.cpp:862
FEngineLoop::Exit()                                        LaunchEngineLoop.cpp:5098
GuardedMain()                                              Launch.cpp:204
```

There are **no PinWright or game frames in the stack** — the defect is engine
shutdown ordering, but PinWright is what puts the engine into the failing state
by requesting exit without ending PIE first.

## Root cause

With PIE still running at exit, nothing tears the session down before Slate goes
away. `FSlateApplication::Shutdown()` destroys the window tree, which destroys
`FAssetEditorViewportLayout`, whose destructor drives `OnLayoutDestroyed()` into
`UEditorEngine::EndPlayMap()`. `EndPlayMap()` sweeps PIE-package objects and calls
`GEditor->GetEditorSubsystem<UAssetEditorSubsystem>()->CloseAllEditorsForAsset(Object)`
(verified in UE 5.8 source at `PlayLevel.cpp:437`). By that point in
`FEngineLoop::Exit()` the `UAssetEditorSubsystem`'s internal state is already torn
down, so `FindEditorsForAssetAndSubObjects` dereferences dead state — a member read
at offset `0x60` off a null/dangling pointer.

Pure teardown-ordering race: `EndPlayMap()` is being run from a Slate destructor
*during* shutdown instead of from the editor tick while the world is still alive.

## Repro

1. `editor.play` (any level).
2. Do anything, or nothing.
3. `editor.quit` (with `discard:true` if packages are dirty).
4. The call returns success (`dirtyCount: 0`), then the process dies with the AV
   above instead of exiting cleanly.

## Impact

- **Critical** by the impact rubric: it is an editor crash, not a wrong result.
- **Silent from the RPC's perspective** — quit acks success, so an automation flow
  believes shutdown was clean. The crash is only visible out-of-band (crash
  reporter, `Saved/Logs`, or the process exit code).
- The crash makes the exit *unclean*, which arms the "Restore Packages" modal on
  the next launch — the known invisible-game-thread-block failure (port LISTENING,
  MCP connection refused) unless the next launch passes `-unattended`. One crashed
  quit cascades into a wedged next session.
- `editor.play` → work → `editor.quit` is a normal automation shape, so the trigger
  is not an exotic edge path.

## Fix (implemented)

`Source/PinWright/Private/Handlers/Editor/EditorQuitHandler.cpp` — after the
existing dirty-package disposition and before scheduling the engine exit:

1. Detect an active session with `GEditor->IsPlaySessionInProgress()` (this also
   covers a session merely *queued* to start on the next tick, which would
   otherwise come up mid-shutdown).
2. If active, call `GEditor->RequestEndPlayMap()` and **hold the RPC response**
   via `Ctx.MakeAsyncToken()` instead of acking immediately. Ending PIE is
   asynchronous — `RequestEndPlayMap()` only queues, `EndPlayMap()` runs on a
   later editor tick — so the exit cannot be issued from the same tick.
3. A 0.1 s `FTSTicker` poll waits until `IsPlaySessionInProgress()` is false. It
   re-issues `RequestEndPlayMap()` if the session went live in the meantime
   (`PlayWorld` set, `ShouldEndPlayMap()` false), because `RequestEndPlayMap()`
   no-ops while `PlayWorld` is null.
4. Once the session is gone, the held response is sent (now carrying
   `pieWasActive` and `pieStopped`) and the existing ~1 s deferred
   `RequestEngineExit` is scheduled as before.
5. If PIE has not stopped within 15 s, the handler **fails loudly** with
   `PIE_STOP_FAILED` and does **not** exit — exiting into a known crash is worse
   than refusing to exit.

The no-PIE path is unchanged apart from the two new reporting fields
(`pieWasActive: false`, `pieStopped: false`).

**Not yet compiled** — the change is C++ and was left in the working tree; a build
plus a live `editor.play` → `editor.quit` run is required to verify.

## See also

`B-editor-quit-crash-open-asset-editors` — a *different* shutdown-ordering crash on
the same RPC (open asset-editor tabs, `~FStaticMeshEditor`, address `0x78`, UE 5.7).
Same family (quit hands the engine a state its shutdown order cannot survive),
different trigger and different fix; both mitigations belong in the quit path.

## History
- `#1-pie-active-quit-crash` `OPEN` reporter — Filed from two live crashes in one session (UE 5.8, PDS project): `editor.quit` with a PIE session running returns a clean success ack and then dies during `FEngineLoop::Exit()` with `EXCEPTION_ACCESS_VIOLATION` at `0x60` in `UAssetEditorSubsystem::FindEditorsForAssetAndSubObjects`, reached from `EndPlayMap()` driven by a Slate viewport-layout destructor. Engine-side teardown-ordering defect, PinWright-triggered by requesting exit without ending PIE first.
- `#2-end-pie-before-exit` `IN-REVIEW` developer — "Changed `EditorQuitHandler.cpp` to end PIE before requesting exit: `GEditor->IsPlaySessionInProgress()` gate, `RequestEndPlayMap()`, response held via `Ctx.MakeAsyncToken()` while a 0.1 s ticker polls for the session to be fully gone (re-issuing the end request if the session was still only queued), then the existing deferred `RequestEngineExit`. Response now reports `pieWasActive`/`pieStopped`; a session that will not stop within 15 s returns `PIE_STOP_FAILED` and cancels the exit rather than crashing. Uncompiled — needs a build and a live `editor.play` → `editor.quit` run to verify."
