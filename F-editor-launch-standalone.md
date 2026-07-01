---
id: F-editor-launch-standalone
title: "Add `editor.launch_standalone` to spawn a `-game` process for multiplayer/standalone test loops"
status: DONE
severity: Low
category: feature
tags: [editor, pie, standalone, multiplayer]
---

# Add `editor.launch_standalone` to spawn a `-game` process for multiplayer/standalone test loops

The play/PIE surface only exposes in-process PIE: `editor.play` (PIEHandler.cpp:42, "Start a Play-In-Editor session") and `ui.runtime.start_session` (the near-duplicate noted in the generated reference at line 10837). There is no MCP-native way to launch a separate standalone client process (`-game`) against the current project. Grepping the handler tree for `launch_standalone` / `LaunchStandalone` / `RequestPlaySession` returns no matches; the only "standalone-ish" path is `build.pipeline.*` cooked-build output, which is not the same workflow.

This blocks two real flows:

- **Multiplayer test loops** where PIE single-process (1 client + listen server in one editor) fails to reproduce a bug that only surfaces with a fully separate client connecting to the editor's listen server.
- **Editor-out-of-loop performance probes** where the editor's tick overhead distorts the measurement, and the agent wants the same map run in `-game` for a clean baseline.

The workaround is `python.execute` invoking `unreal.SystemLibrary.execute_console_command("LaunchStandalone")` or shelling out via host OS — both work but lose typed params (map, extra args) and the structured success/PID return value.

**Fix:** New handler `editor.launch_standalone` in `Handlers/Editor/PIEHandler.cpp` (or a sibling file). Params:

- `map` (optional string) — asset path to a level, defaults to the editor's current world.
- `extraArgs` (optional string) — passed through to the spawned process command line (e.g. `"-windowed -resx=1280"`).
- `numClients` (optional int, default 1) — for the multiplayer case, spawn N client processes plus optionally a listen server.
- `listenServer` (optional bool, default false) — when true, the first process is launched with `?listen` and subsequent clients connect to it.

Implementation walks `GEditor->RequestPlaySession` with `FRequestPlaySessionParams::StandaloneCount` (UE 5.4+), or falls back to spawning `FPlatformProcess::CreateProc` against the current `FPaths::GetProjectFilePath()` with `-game` and the requested args. Returns `{ pid: <int>, commandLine: "..." }` per spawned process. No mutation of the editor world.

## History
- `#1-feature-request` `OPEN` reporter — Only PIE is exposed (`editor.play`, `ui.runtime.start_session`); no `editor.launch_standalone` to spawn a `-game` process. Blocks multiplayer test loops where PIE's single-process model fails to reproduce client-server bugs, and clean-baseline perf probes where editor tick distorts measurements. Workaround is `python.execute("LaunchStandalone")` or host-OS shell. Severity Low — workarounds exist and the use case is niche.
- `#2-implemented-launch-standalone` `IN-REVIEW` developer — Added `editor.launch_standalone` in `Handlers/Editor/EditorLaunchHandler.cpp` plus internal header for the pure command-line builder. Uses `FPlatformProcess::CreateProc(ExecutablePath, ...)` per instance with PID returned via OutProcessID; supports numClients[1..8], listenServer (instance 0 gets ?listen, others get 127.0.0.1), and extraArgs passthrough. RequestPlaySession path was dropped (UE 5.6 FRequestPlaySessionParams has no StandaloneCount/bLaunchStandalone — multi-client launch is GEditor-internal). Tests in `Tests/EditorOps/TestEditorLaunchHandler.cpp` cover the builder; counterfactual: reverting the listen-server branch makes instance 0's command line lack ?listen.
- `#3-skip-editor-offline` `SKIP` tester — Editor RPC port 19880 not listening (Test-NetConnection False; only TIME_WAIT entries from earlier sessions). Cannot exercise `editor.launch_standalone?` schema fetch or live spawn via MCP. Static review of `EditorLaunchHandler.cpp`, `EditorLaunchHandlerInternal.h`, and `Tests/EditorOps/TestEditorLaunchHandler.cpp` matches the IN-REVIEW claims (registration with map/extraArgs/numClients/listenServer params, CreateProc with OutProcessID, instance-0-gets-?listen / instance-N-gets-127.0.0.1 branch, three unit tests), but live verification deferred until the editor is running.
- `#4-verify-live-spawn` `DONE` tester — Verified live via direct HTTP /rpc. (1) `editor.launch_standalone?` returns full schema with map/extraArgs/numClients/listenServer params and clamp note [1,8]. (2) `numClients=1 extraArgs="-windowed -resx=320 -resy=240"` returned `success:true` and a real PID with commandLine containing `-game` and the extraArgs verbatim against the editor's current world (`L_Core`). (3) `numClients=2 listenServer=true` returned two PIDs; instance 0 commandLine has `L_Core?listen`, instance 1 has `127.0.0.1` — counterfactual from #2 confirmed. All three spawned PIDs killed with taskkill afterwards.
