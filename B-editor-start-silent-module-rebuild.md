---
id: B-editor-start-silent-module-rebuild
title: "editor_start with a map (unattended) silently runs Build.bat/UBT on stale project modules and reports only success, compiling other agents' in-progress C++"
status: OPEN
severity: Medium
category: bug
tags: [editor_start, editor-lifecycle, ubt, build, unattended, modules-out-of-date, side-effect, shared-checkout]
encounters: 1
lastSeen: 2026-09-23T20:55:00Z
---

# Starting the editor can build the project, and the caller is never told

UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. `mcp__pinwright__editor_start {map: /Game/System/FrontEnd/Maps/L_Core, unattended_script: true}`
returned `{success: true, pid, commandLine, elapsedSeconds: 124.4}`. `Saved/Logs/PDS.log` shows that during
that start the editor ran `Build.bat Development Win64 -Project=... -TargetType=Editor` (UBT, 24 actions,
`Result: Succeeded`, 46 s), recompiling `Plugins/App/Source/App/Api/*` and `Acceptance/*`. Other agents
were editing those files in the same checkout at the time, and the task rules said not to build. The
rebuild happens because the engine's "modules are out of date, rebuild?" prompt is auto-accepted under
`-RunningUnattendedScript`. The response gives no hint that a build ran.

Why it matters: in a shared checkout, a start that only means "restart the editor" can compile someone
else's half-finished code into the loaded DLL. A failing build would make the start look like a hang or a crash.

**Fix:** detect stale modules before spawning (compare module timestamps, or pass `-NoCompile`/equivalent
so the editor refuses rather than builds). Report `modulesRebuilt: true` plus the UBT result in the
response, and offer an explicit `allow_build` flag (default false for unattended starts).

## History
- `#1-start-rebuilt-app-module` `OPEN` reporter - UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Noticed after the fact in the log; the build succeeded, so no work was lost, but the no-build rule was broken without anyone deciding to break it.
