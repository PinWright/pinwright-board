---
id: B-console-verbs-bypass-typed-verb-guards
title: "editor.console_command and system.console_command guard only scalability CVars, so QUIT_EDITOR, PY, EXECFILE and DEBUG CRASH-family lines bypass the typed verbs' safety checks"
status: IN-REVIEW
severity: High
category: bug
tags: [console, console-command, editor-quit, python, crash, safety, shared-editor, multi-agent, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# Console verbs bypass the safety checks of the typed verbs

`editor.console_command` (`Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp:298-313`)
and `system.console_command` (`Source/PinWright/Private/Handlers/System/SystemControlHandler.cpp:1151-1183`)
refuse exactly one class of line: a scalability CVar set, through
`ScalabilityConsoleGuard::IsScalabilityPinningLine` (`Source/PinWright/Private/Handlers/ScalabilityConsoleGuard.h`).
Every other line goes to `GEditor->Exec` / `GEngine->Exec` (plugin HEAD `71c91649`). Several engine
commands reach the same effects that typed PinWright verbs deliberately gate:

1. **`QUIT_EDITOR`** reaches `UUnrealEdEngine::CloseEditor`
   (`Engine/Source/Editor/UnrealEd/Private/EditorServer.cpp:5993-5997`,
   `Engine/Source/Editor/UnrealEd/Private/UnrealEdEngine.cpp:943-958`): end PIE, then
   `RequestEngineExit`. `editor.quit` (`Source/PinWright/Private/Handlers/Editor/EditorQuitHandler.cpp:141`)
   refuses with `EDITOR_IN_USE` when another client drove the editor in the last 5 minutes, refuses
   with `UNSAVED_CHANGES` on dirty user packages, closes asset editors (one left open crashes the
   editor in its destructor during shutdown), and terminates running jobs so streaming clients get
   a terminal event. The console route skips all of it: unsaved work is lost and another agent's
   session dies mid-call, the scenario `ClientActivity` was built to prevent.
2. **`PY <code>`** runs through the PythonScriptPlugin exec handler
   (`Engine/Plugins/Experimental/PythonScriptPlugin/Source/PythonScriptPlugin/Private/PythonScriptPlugin.cpp:1060`).
   Both console verbs are in the same SafePoint deferral list as `python.execute`
   (`Source/PinWright/Private/Dispatch/SafePoint.cpp:170-172`), so stack position is equal. What is
   lost is the rest of `python.execute`
   (`Source/PinWright/Private/Handlers/System/PythonExecuteHandler.cpp`): the private scope with
   `sys.modules` restore, captured log output in the response, the PIE-active sentinel warning, and
   the tracker that reports tick or shutdown callbacks the script leaked.
3. **`EXECFILE <path>`** (`EditorServer.cpp:329-338`) runs every line of a file through `Exec`.
   Only the outer `EXECFILE` line is checked, so a file can pin scalability CVars or carry
   `QUIT_EDITOR` past both guards.
4. **`DEBUG <sub>`** routes to `UEngine::HandleDebugCommand` then `PerformError` /
   `PerformBlockingError` (`Engine/Source/Runtime/Engine/Private/UnrealEngine.cpp:5903-5906`,
   `:8809-8819`, `:11213-11800`), compiled into every non-Shipping build, editor included. Its
   subcommands exist to kill or wedge the process: `CRASH`, `GPF`, `CHECK`, `FATAL`,
   `ENSURE`, `RENDERCRASH`, `THREADCRASH`, `GPUCRASH`, `TERMINATE`, `ABORT`, `SOFTLOCK`,
   `INFINITELOOP`, `STALL`, `HITCH`, `SPIN`, `RECURSE`, `EATMEM`, `BUFFEROVERRUN` and more. In a
   shared editor that is every client's session.

Dropped from the gap-analysis claim after checking source: `MACRO` / `EXEC` do not run command
files in the editor. `UEditorEngine::SafeExec` (`EditorServer.cpp:325-327`) opens a "Tried to execute
deprecated command" message dialog, which `PinWrightAutomationMode` suppresses to its default during
dispatch (`Source/PinWright/Private/Dispatch/ScopedUnattendedRpc.h`), so they are no-ops here.

Engines: all supported (5.3 to 5.8).

**Fix:** a sibling of `ScalabilityConsoleGuard.h` (for example `ConsoleCommandGuard.h`) applied in
both handlers, before `Exec`, that matches the first token (case-insensitive, `FParse::Command`
semantics) and refuses:
`QUIT_EDITOR` and `CLOSE_SLATE_MAINFRAME` with a code pointing at `editor.quit`;
`PY` with a code pointing at `python.execute`; `EXECFILE` with a code saying to send the lines
individually; `DEBUG` followed by a crash, hang or memory subcommand with a code that names the
effect. Keep `force:true` as the explicit override, matching the scalability guard, and extend its
param doc. Tests: one per refused token plus a mixed-case line and a leading-whitespace line.

**Related:** `B-console-command-sg-cvar-pin-freezes-scalability` and
`B-console-member-cvar-pin-freezes-scalability` (origin of the existing guard),
`B-python-execute-reentrant-gc-crash`.

## History
- `#1-console-guard-gaps` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified at plugin HEAD `71c91649`: both console handlers refuse only `IsScalabilityPinningLine`. Engine routes confirmed on 5.8: `QUIT_EDITOR` to `CloseEditor` then `RequestEngineExit`, `PY` to PythonScriptPlugin, `EXECFILE` to `ExecFile`, `DEBUG` to `PerformError`. Two claims from the analysis were corrected: console `PY` does share `python.execute`'s SafePoint deferral (what it skips is the rest of that handler), and `MACRO`/`EXEC` are deprecated no-ops in the editor. Severity High: data loss and killing other clients' sessions sit in the Critical class, bumped down one because reaching them takes a deliberate raw console line.
- `#2-console-command-guard` `IN-REVIEW` developer - Added `Source/PinWright/Private/Handlers/ConsoleCommandGuard.h` (`ConsoleCommandGuard::FindRefusal`), called right after the scalability guard in `editor.console_command` (`Handlers/Editor/EditorCommandHandler.cpp`) and `system.console_command` (`Handlers/System/SystemControlHandler.cpp`), skipped by `force:true`. The first command word is matched with `FParse::Command` itself after `TrimStart()`, so case, leading whitespace and word boundaries (`py.foo`, `DEBUG CRASH_now`) behave exactly as the engine's exec handlers do. Refusals: `QUIT_EDITOR` / `CLOSE_SLATE_MAINFRAME` -> `EDITOR_QUIT_USE_TYPED_VERB` (`useVerb: editor.quit`); `PY` -> `PYTHON_USE_TYPED_VERB` (`useVerb: python.execute`); `EXECFILE` -> `EXECFILE_SEND_LINES_INDIVIDUALLY`; `DEBUG` + a 5.3-5.8 `PerformError` / `PerformBlockingError` subcommand -> `DEBUG_COMMAND_CRASHES_PROCESS` / `DEBUG_COMMAND_HANGS_PROCESS` / `DEBUG_COMMAND_EXHAUSTS_MEMORY`; error data carries `refusedCommand`, `useVerb`, `forceOverrides`. Allowed: `DEBUG HITCH` / `RENDERHITCH` / `RESETLOADERS` / `LONGLOG`, `MACRO` / `EXEC`, and `EXIT` / `QUIT`, verified not to be editor-quit aliases on this route (only `ULocalPlayer::Exec_Editor` handles them, `LocalPlayer.cpp:1620`, and it ends PIE). `SPIN` / `RENDERSPIN` are refused as hangs per this ticket although they are bounded like `HITCH`. Six `ERR_*` constants in `Handlers/ErrorCodes.h`; wiki `docs/wiki-src/editor.md` and `system.md` and `docs/error-code-catalog.md` updated. Tests in `Tests/EditorOps/TestConsoleCommandGuard.cpp`: `PinWright.core.console_command_guard.{RefusesEachBypassToken,CaseAndWhitespaceVariants,AllowsOrdinaryAndProfilingLines}` on the predicate (QUIT/DEBUG only there, since a reverted guard would kill the suite editor) and `PinWright.{system,editor}.console_command.TypedVerbBypassRefusedWithoutForce` on both handlers (EXECFILE of a missing file, `PY pass`, force, ordinary cvar read). Three files compile-checked with `-SingleFile` on 5.8; suite not run.
