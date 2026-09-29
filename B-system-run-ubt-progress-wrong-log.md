---
id: B-system-run-ubt-progress-wrong-log
title: "system.run_ubt progress heartbeat watches Engine/Programs/UnrealBuildTool/Log.txt, which installed engines never write, so ubtLogBytes is always absent on launcher installs"
status: WONTFIX
severity: Low
category: bug
tags: [system, ubt, run_ubt, jobs, progress, installed-engine, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# system.run_ubt progress reads the wrong UBT log on installed engines

`system.run_ubt` has no output pipe, so its only mid-run signal is a 10 s heartbeat that reports
`elapsedSeconds` plus `ubtLogBytes`, the size of UBT's log, as evidence the build is moving
(`Source/PinWright/Private/Handlers/System/SystemControlHandler.cpp:974-1000`, plugin HEAD
`71c91649`). The path is hardcoded as `FPaths::EngineDir()/Programs/UnrealBuildTool/Log.txt`
(`:981-982`).

UBT writes its default log to `Unreal.EngineProgramSavedDirectory/UnrealBuildTool/Log.txt`
(`Engine/Source/Programs/UnrealBuildTool/Modes/BuildMode.cs:138`). That directory is
`Engine/Programs` only for source builds; for an installed engine it is the user settings directory
(`Engine/Source/Programs/Shared/EpicGames.Build/Unreal.cs:213-214` on 5.8, `:165` on 5.3, `:158` on
5.5), which on Windows is the per-user local application data folder. Every engine a launcher
installs (`Engine/Build/InstalledBuild.txt` present) takes that branch.

Result on launcher installs: the file under `Engine/Programs` does not exist, `FileSize` returns -1,
and the heartbeat drops `ubtLogBytes` for the whole run. A caller watching a multi-minute build sees
only a clock and cannot tell a compiling build from a stuck one. If a stale file does exist there
(for example from an earlier source build of the same tree), the heartbeat reports a constant size,
which reads as "stalled".

Engines: all supported (5.3 to 5.8) when installed from the launcher.

**Fix:** pass an explicit `-log="<path>"` to UBT (the `-Log` option,
`Engine/Source/Programs/UnrealBuildTool/GlobalOptions.cs:33`) pointing into the project's
`Saved/` directory, and watch that file. That also gives the verb a log it can return on completion,
which is the cheap half of the missing stdout/stderr capture in
`B-registration-summaries-promise-absent-fields`.

**Related:** `B-system-run-ubt-missing-target-project`, `B-system-run-ubt-live-coding-unexplained`.

## History
- `#1-wrong-log-path` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified from source at plugin HEAD `71c91649`: heartbeat path `SystemControlHandler.cpp:981-982` versus UBT's `EngineProgramSavedDirectory` (`Unreal.cs:214`, user settings dir when `IsEngineInstalled()`) used by `BuildMode.cs:138`. All six local engines 5.3 to 5.8 carry `Engine/Build/InstalledBuild.txt`. Severity Low: a progress field is missing, the job still completes and reports its exit code; rare reach.
- `#2-verb-removed` `WONTFIX` developer - Obsolete: `system.run_ubt` was deleted outright (no shim or alias) by task 11 of plan `Plugins/PinWright/tasks/2026-09-28-gap-quick-wins.md`, together with `Handlers/BuildTools/ProcPollBind.h` and `UbtEntryPoint.h`. The verb could not build the editor target it runs in; agents now build from their own shell with the editor closed (`Build.bat`/`Build.sh ... -TargetType=Editor`) or use `system.live_coding_compile` for a running editor. No code remains for this defect to apply to.
