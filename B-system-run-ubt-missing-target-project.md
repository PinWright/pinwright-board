---
id: B-system-run-ubt-missing-target-project
title: "system.run_ubt builds an invalid UBT command line: no target when `target` is omitted, no `-project` when it is given"
status: WONTFIX
severity: Medium
category: bug
tags: [system, ubt, build, run_ubt, command-line, installed-engine, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# system.run_ubt builds an invalid UBT command line

`system.run_ubt` cannot build the host project with either shape of call. Both failures surface only
as `UBT_NONZERO_EXIT` with `{exit_code}`, because the verb captures no output (see
`B-registration-summaries-promise-absent-fields`, item 2), so the caller never sees UBT's reason.

Argument assembly, `Source/PinWright/Private/Handlers/System/SystemControlHandler.cpp:916-959`
(plugin HEAD `71c91649`):

1. **`target` omitted.** The `target` param doc (`:895`) says it "Defaults to the current editor
   target", but the code (`:920-930`) only appends `-project="<uproject>"`, giving
   `-project="..." Win64 Development`. UBT has no target name and no `-TargetType=`, so
   `TargetDescriptor.ParseCommandLine` throws "No target name was specified on the command-line."
   (`Engine/Source/Programs/UnrealBuildTool/Configuration/Descriptors/TargetDescriptor.cs:561-564`
   on 5.8; `Configuration/TargetDescriptor.cs:508` on 5.3, `:556` on 5.5). The documented default
   call fails every time.
2. **`target` given.** The `else` branch means `-project` is never added when a target is named
   (`:920-923`). Without `-project`, UBT resolves the target only through
   `NativeProjects.TryGetProjectForTarget` (`TargetDescriptor.cs:497`), which knows projects listed
   under the engine's `.uprojectdirs` roots. A project outside the engine tree, the normal layout
   with a launcher-installed engine, fails with "Couldn't find target rules file for target ..."
   (`Engine/Source/Programs/UnrealBuildTool/Configuration/Rules/RulesAssembly.cs:937`).

Workaround: pass `target` and put `-project="<path>"` in `additionalArgs` yourself. Nothing tells the
caller to do this.

Engines: all supported (5.3 to 5.8); same UBT behavior on each.

**Fix:** always pass `-project="<FPaths::GetProjectFilePath()>"` when the editor has a project. When
`target` is omitted, add `-TargetType=Editor` (UBT resolves the project's editor target from it,
`TargetDescriptor.cs:542-558`), or resolve the name explicitly, so the documented default holds. Add
a test that asserts the assembled argument string for both shapes.

**Related:** `B-system-run-ubt-live-coding-unexplained` and `B-system-run-ubt-progress-wrong-log`
(same verb, separate defects), `B-registration-summaries-promise-absent-fields` (no stdout/stderr
capture, which is why the UBT error text is invisible).

## History
- `#1-invalid-command-line` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified from source at plugin HEAD `71c91649`: `SystemControlHandler.cpp:918-930` appends either the target or `-project`, never both. No target gives `-project=... Win64 Development`, which UBT rejects at `TargetDescriptor.cs:563`; a target without `-project` only resolves for native projects (`TargetDescriptor.cs:497`, `RulesAssembly.cs:937`). Grouped as one ticket because both are the same argument builder and one fix covers them. Severity Medium: a documented default that always fails, with a workaround that needs a source dive; the failure is not silent (non-zero exit), so not High.
- `#2-verb-removed` `WONTFIX` developer - Obsolete: `system.run_ubt` was deleted outright (no shim or alias) by task 11 of plan `Plugins/PinWright/tasks/2026-09-28-gap-quick-wins.md`, together with `Handlers/BuildTools/ProcPollBind.h` and `UbtEntryPoint.h`. The verb could not build the editor target it runs in; agents now build from their own shell with the editor closed (`Build.bat`/`Build.sh ... -TargetType=Editor`) or use `system.live_coding_compile` for a running editor. No code remains for this defect to apply to.
