---
id: B-system-run-ubt-live-coding-unexplained
title: "system.run_ubt on the running editor's own target fails while Live Coding is active, and the verb neither detects it up front nor explains the bare non-zero exit"
status: WONTFIX
severity: Medium
category: bug
tags: [system, ubt, run_ubt, live-coding, build, error-message, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# system.run_ubt fails without a reason under an active Live Coding session

The registration summary of `system.run_ubt` sells it as "Useful for scripted hot-recompile"
(`Source/PinWright/Private/Handlers/System/SystemControlHandler.cpp:893`, plugin HEAD `71c91649`).
The hot-recompile case is building the editor target of the editor that is serving the call. When
Live Coding is active for that editor, UBT refuses before compiling anything:

`HotReload.CheckForLiveCodingSessionActive` throws "Unable to build while Live Coding is active. Exit
the editor and game, or press Ctrl+Alt+F11 if iterating on code in the editor or game"
(`Engine/Source/Programs/UnrealBuildTool/System/HotReload.cs:269-278` on 5.8; `:248` on 5.3, `:262`
on 5.5). It keys the check on a named mutex derived from the target executable, so it fires for
exactly this call.

What the caller gets: a job that fails with `UBT_NONZERO_EXIT` and `{exit_code}`
(`Source/PinWright/Private/Handlers/BuildTools/ProcPollBind.h:37-41`). The UBT message is not
captured (no output pipe, see `B-registration-summaries-promise-absent-fields` item 2), and the
handler (`SystemControlHandler.cpp:893-1001`) never consults Live Coding state before spawning. An
agent has no way to learn the reason short of reading UBT's log on disk, and the fix it needs is a
different verb.

The plugin already has that verb and the state probe: `system.live_coding_status` and
`system.live_coding_compile` (`Source/PinWright/Private/Handlers/System/LiveCodingHandler.cpp:112`,
`:163`), whose own summary contrasts itself with `system.run_ubt`.

Engines: all supported Win64 editors where Live Coding is enabled for the session (Live Coding is
Windows-only).

**Fix:** before spawning, when the requested (or defaulted) target is the running editor's target
and `ILiveCodingModule` reports the session started, refuse with a dedicated code (for example
`LIVE_CODING_ACTIVE`) whose message names `system.live_coding_compile` as the in-editor path and
says an external build needs the editor closed. As a backstop, recognise the Live Coding line in
the UBT log (see `B-system-run-ubt-progress-wrong-log` for giving the verb a log of its own) and map
the failure to the same code.

**Related:** `F-live-coding-trigger` (added the live_coding verbs), `B-live-coding-mutex-blocks-cross-project-build`
(the same UBT check misfiring across projects that share one editor executable),
`B-system-run-ubt-missing-target-project`.

## History
- `#1-live-coding-unexplained` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified from source at plugin HEAD `71c91649`: UBT throws at `HotReload.cs:277` when a Live Coding session is active for the target; `system.run_ubt` (`SystemControlHandler.cpp:893-1001`) has no Live Coding check and reports only `{exit_code}` (`ProcPollBind.h:37-41`), although `system.live_coding_status` exists (`LiveCodingHandler.cpp:112`). Severity Medium: the call fails honestly (non-zero exit) but the reason is hidden and the remedy needs outside knowledge; reach limited to in-editor rebuild attempts.
- `#2-verb-removed` `WONTFIX` developer - Obsolete: `system.run_ubt` was deleted outright (no shim or alias) by task 11 of plan `Plugins/PinWright/tasks/2026-09-28-gap-quick-wins.md`, together with `Handlers/BuildTools/ProcPollBind.h` and `UbtEntryPoint.h`. The verb could not build the editor target it runs in; agents now build from their own shell with the editor closed (`Build.bat`/`Build.sh ... -TargetType=Editor`) or use `system.live_coding_compile` for a running editor. No code remains for this defect to apply to.
