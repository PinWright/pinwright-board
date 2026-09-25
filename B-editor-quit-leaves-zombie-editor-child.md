---
id: B-editor-quit-leaves-zombie-editor-child
title: "An editor started by editor_start / editor_restart stays a zombie (<defunct>) under mcp_proxy.py after editor.quit; its Engine/Intermediate/EditorRuns/<pid> marker then crashes UBT's hot-reload check, so the next Build.sh fails until -NoHotReloadFromIDE"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, editor_start, editor_restart, editor.quit, mcp-proxy, zombie, waitpid, child-process, linux, ubt, hot-reload, build]
encounters: 1
lastSeen: 2026-09-25T11:48:00Z
---

# Editors spawned by the MCP proxy are never reaped

## Symptom

Linux, UE 5.8, PDS (`unreal-fpv`). `editor_restart` spawned the editor (PID 119357, parent
`Plugins/PinWright/Content/Python/mcp_proxy.py` PID 3921097). After `editor.quit` (success, PIE stopped) the process
stayed `State: Z (zombie)` indefinitely; the proxy never `wait()`s its child. `ps` on the same box lists nine
`[UnrealEditor] <defunct>` processes, all children of `mcp_proxy.py` instances of this and other checkouts
(`unreal-fpv-wt2`), the oldest from Sep 23.

Consequences seen in this session:

1. `Build.sh PDSEditor Linux Development ...` failed in 18 s with `Result: Failed (OtherCompilationError)`:
   `ArgumentException: The value cannot be an empty string. (Parameter 'path')` at
   `UnrealBuildTool.HotReload.ShouldDoHotReloadFromIDE` (`HotReload.cs:494`). UBT walks
   `Engine/Intermediate/EditorRuns/<pid>` markers; a zombie still exists as a process, so the marker is not treated
   as stale, and its `Filename` is empty (no `/proc/<pid>/exe`). Here the marker was a wt2 zombie's (4170836),
   so one agent's un-reaped editor breaks every other checkout's build on the shared engine.
2. The next editor of the same checkout wrote `Saved/Logs/PDS_2.log` instead of `PDS.log`, so log greps that assume
   `PDS.log` read the previous session.

## Expected

The proxy (or whatever spawns the editor for `editor_start` / `editor_restart`) reaps the child when it exits
(a `wait`/`poll` on exit, or spawn it detached with a double fork / `start_new_session` so init reaps it).

## Workaround

Build with `-NoHotReloadFromIDE` while no editor of the checkout is running. Zombies of other sessions cannot be
cleared without touching their proxy.

## History
- `#1-zombie-editors-break-ubt` `OPEN` reporter - Hit while rebuilding PDS for QA #1044 (acceptance row 1a). Needed the `-NoHotReloadFromIDE` workaround for both builds of the session.
- `#2-reaper-thread-per-child` `IN-REVIEW` developer - Root cause: `Proxy._editor_start` spawns the editor with `subprocess.Popen` (own session) and only polls or waits until the verb returns. Nothing waits on a child that is still running at that point, and CPython reaps a discarded Popen only at the proxy's next Popen call, so an editor that later quits stays `<defunct>` under the long-lived proxy. Fix: new `_reap_on_exit(proc)` in `Content/Python/mcp_proxy.py` starts a daemon thread calling `proc.wait()` for a child still running when `_editor_start` returns (this covers `editor_restart`, which calls it, and the ready, timeout and interrupted paths). The same helper covers the macOS/xdg `open` child of `_open_uproject`. The proxy never blocks on it, a daemon thread never delays a proxy exit, and the editor still survives a proxy exit or restart. `wait='exit'` already reaped its child through `proc.wait()`. `editor_prepare_tests` spawns nothing. Test: `ReapSpawnedChildTest` (`Content/Python/tests/test_mcp_proxy_editor_start.py`) spawns a real child through `_editor_start`. It checks that the child is still running after the verb and after `request_shutdown`, and that it is gone from `/proc`, not Z, once it exits. The test fails without the fix (child stuck in `Z`). Full Python suite 260 OK (2 skipped). Live check (Linux, display :0): a standalone fixed proxy ran `editor_start {extra_args:[-SKIPCOMPILE]}`. The editor was pid 333678 under proxy 333676 and ready in 94 s, and `EditorRuns` listed `333678`. `editor.quit` succeeded. 20 s later `/proc/333678` was gone (not Z), the proxy was still alive, and the `333678` marker was removed. That editor logged to `PDS.log`, not `PDS_2.log`. A second start, pid 339842, kept running after its proxy exited (reparented to init). Other sessions' old proxies keep their zombies until those proxies restart on this code. Plugin commit `8fcc0b2a`.
