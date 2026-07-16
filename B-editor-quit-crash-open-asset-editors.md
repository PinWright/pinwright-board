---
id: B-editor-quit-crash-open-asset-editors
title: "editor.quit crashes the editor (EXCEPTION_ACCESS_VIOLATION in ~FStaticMeshEditor) when asset-editor windows are still open — shutdown must close asset editors first"
status: OPEN
severity: High
category: bug
tags: [editor-quit, shutdown, crash, access-violation, static-mesh-editor, asset-editor, unclean-exit, restore-packages]
encounters: 1
lastSeen: 2026-07-16T11:09:25+03:00
---

# `editor.quit` crashes the editor when asset-editor windows are still open

`editor.quit` returned a clean `{"requested":true,"reason":"mcp.editor.quit","saved":false,"discarded":false,"dirtyCount":0}` and then the editor process died with an unhandled `EXCEPTION_ACCESS_VIOLATION reading 0x0000000000000078` during engine shutdown (UE 5.7, PDS project, 2026-07-16). A Static Mesh Editor tab was open at quit time.

## Crash anatomy (from the user-captured callstack)

Bottom-up: `FEngineLoop::Exit` → `FSlateApplication::Shutdown` → `CloseAllWindowsImmediately` → `~SWindow` widget-tree teardown → `SStandaloneAssetEditorToolkitHost::ShutdownToolkitHost` (`SStandaloneAssetEditorToolkitHost.cpp:406`) → `FStaticMeshEditor::~FStaticMeshEditor` (`StaticMeshEditor.cpp:264`) → `TMulticastDelegateBase::RemoveAll()` → `FMRSWRecursiveAccessDetector::AcquireWriteAccess` (`MTAccessDetector.h:615`) → AV reading `0x78` (null object + member offset).

Interpretation: this is the classic UE shutdown-order defect — asset editors still open at exit are destroyed late, by Slate window teardown, *after* the objects whose multicast delegates they subscribed to are already gone. The Static Mesh Editor destructor's `RemoveAll()` walks a dangling delegate owner. The engine bug is not ours to fix, but **PinWright's quit path hands the engine exactly the state that triggers it**.

## Why this is High

- Any `editor.quit` issued while the user (or an automation flow, e.g. a mesh/preview inspection) left an asset editor open can crash instead of exiting cleanly. Which editor tabs happen to be open is not something the calling agent can see or control.
- The crash makes the exit *unclean*, which arms the "Restore Packages" modal on the next launch — the known invisible-game-thread-block failure (port LISTENING, MCP connection refused) unless the next launch happens to pass `-unattended`. So one crashed quit can cascade into a wedged next session.
- The tool reports `requested:true` success, so the caller believes shutdown went fine; the crash is only visible out-of-band (crash reporter / logs).

## Proposed fix

In the `editor.quit` handler, before requesting engine exit:

1. Close all open asset editors on the game thread: `GEditor->GetEditorSubsystem<UAssetEditorSubsystem>()->CloseAllAssetEditors()` (this tears each toolkit down through the orderly close path while their delegate owners are still alive), and let one tick/flush pass if needed.
2. Only then proceed with the existing quit request (`RequestEngineExit` / `FPlatformMisc::RequestExit` — whatever the handler uses today).
3. Optionally report `closedAssetEditors: N` in the response for observability.

Repro sketch: open any static mesh in the Static Mesh Editor (double-click `SM_*` in Content Browser), call `editor.quit`, watch the process die in `~FStaticMeshEditor` instead of exiting cleanly. (Crash is timing/ordering dependent but reproduced live on UE 5.7 with one SM editor tab open.)

## History
- `#1-initial-report` `OPEN` reporter — Filed from a live crash: `editor.quit` during the sumo-drone work session killed the editor in `~FStaticMeshEditor` delegate teardown with an SM editor tab open (full callstack captured in this ticket). `dirtyCount:0` so no data was lost, but the unclean exit arms the Restore-Packages modal for the next launch. Proposed mitigation: `CloseAllAssetEditors()` before requesting engine exit.
