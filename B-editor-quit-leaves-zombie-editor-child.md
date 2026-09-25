---
id: B-editor-quit-leaves-zombie-editor-child
title: "An editor started by editor_start / editor_restart stays a zombie (<defunct>) under mcp_proxy.py after editor.quit; its Engine/Intermediate/EditorRuns/<pid> marker then crashes UBT's hot-reload check, so the next Build.sh fails until -NoHotReloadFromIDE"
status: OPEN
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
