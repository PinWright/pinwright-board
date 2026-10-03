---
id: B-live-coding-mutex-blocks-cross-project-build
title: "Building the plugin fails while ANY other UE project is open, because the Live Coding mutex is keyed on the shared UnrealEditor.exe path"
status: IN-REVIEW
severity: Medium
category: bug
tags: [build, live-coding, ubt, installed-engine, tooling, developer-experience]
encounters: 1
costly: 1
lastSeen: 2026-08-13T00:00:00Z
---

# Live Coding mutex blocks builds across unrelated projects

Building this plugin fails with:

```
Unable to build while Live Coding is active. Exit the editor and game,
or press Ctrl+Alt+F11 if iterating on code in the editor or game
Result: Failed (OtherCompilationError)
```

**even when the editor for this project is already closed** — as long as a *different*
UE project's editor is open with Live Coding active.

## Root cause (engine source)

`UnrealBuildTool/System/HotReload.cs`:

- `CheckForLiveCodingSessionActive` (`:270-278`) throws when `IsLiveCodingSessionActive` returns true.
- `IsLiveCodingSessionActive` (`:287-317`) builds the mutex name from **`Makefile.ExecutableFile`** —
  the *output executable path* — not from the project:

```csharp
StringBuilder MutexName = new StringBuilder("Global\\LiveCoding_");
for (int Idx = 0; Idx < Executable.FullName.Length; Idx++) { ... }
```

On an **installed engine**, every content-only project's editor target resolves to the same
executable, `C:\UE_5.8\Engine\Binaries\Win64\UnrealEditor.exe`. So all such projects share one
mutex name and any one of them with Live Coding active blocks builds for all the others.

Observed 2026-08-13: an open `X:\src\unreal\unreal-fpv-dev\DroneFootball.uproject` editor blocked
a build of `X:\src\unreal\EAContentExamples58` whose own editor was already closed.

Note the collision is **only** the mutex. The two projects are otherwise fully isolated — the
DroneFootball editor loads PinWright from its own separate checkout
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Binaries\Win64\`), so it never holds a handle on
this project's DLLs.

## Why the obvious workarounds are wrong

- **Killing `LiveCodingConsole.exe` does not help.** The mutex is created by `LiveCodingModule` inside
  the **editor** process, not by the console. Killing the console leaves the mutex held.
- **Killing the other editor is unacceptable** on a machine that runs editors for several projects —
  it destroys that project's unsaved work.

## Workaround (verified)

Pass `-NoHotReloadFromIDE` to UBT. It sets `bAllowHotReloadFromIDE = false`
(`BuildConfiguration.cs:160-162`), which is the first condition in `CheckForLiveCodingSessionActive`,
so the check is skipped entirely:

```
C:\UE_5.8\Engine\Build\BatchFiles\Build.bat UnrealEditor Win64 Development ^
  -Project=<...>.uproject -WaitMutex -FromMsBuild -NoHotReloadFromIDE
```

Safe for this plugin specifically because the build writes only
`Plugins/PinWright/Binaries/Win64`, which no other project's editor loads, and a content-only
project on an installed engine never relinks `UnrealEditor.exe`. It is **not** safe if the target's
own editor is running — that is what the check exists to prevent.

**Fix:** document `-NoHotReloadFromIDE` in the plugin's build instructions, with the constraint that
the *target* project's editor must still be closed. This is an engine design limitation, not
something the plugin can fix in code.

## Related trap found in the same session

The build also failed once with:

```
LINK : fatal error LNK1104: cannot open file '...\UnrealEditor-PinWright.dll'
```

after **all 107 TUs compiled cleanly**. The linker had already **deleted** both
`UnrealEditor-PinWright.dll` / `.pdb` and `UnrealEditor-PinWrightGeometry.dll` / `.pdb` before
failing to recreate them, leaving the plugin with no binaries at all. Causes seen: a leftover
`UnrealEditor-Cmd.exe` from an automation run still holding the DLLs. Re-running the build after the
holder exits links cleanly. **Back up `Binaries/Win64` before building**, and never trust
`Build.bat`'s shell exit code — it reported success on a run whose output ended
`Result: Failed`.

## History
- `#1-initial-repro` `OPEN` reporter — "UBT refuses to build while any OTHER UE project's editor has Live Coding active, because HotReload.cs:287 keys the Global\\LiveCoding_ mutex on the shared installed-engine UnrealEditor.exe path rather than the project. Killing LiveCodingConsole.exe does not release it (the editor holds it). Workaround: -NoHotReloadFromIDE, safe only when the target project's own editor is closed. Also recorded the LNK1104 trap where the linker deletes both DLLs+PDBs before failing."
- `#2-run-ubt-removed` `OPEN` developer - Note on the `system.run_ubt` removal: this ticket never depended on that verb (it is about an external `Build.bat` run hitting the `Global\LiveCoding_` mutex), so its substance is unchanged. What changes is scope: the plugin no longer offers any in-editor way to spawn UBT, so the external build this ticket describes is now the only full-build route, and `system.live_coding_compile` is the only in-editor one. The proposed fix (document `-NoHotReloadFromIDE` with the target-editor-closed constraint) appears in the plugin `CLAUDE.md` "Building" section, but no developer has claimed it against this ticket, so status is left OPEN.
- `#3-already-resolved-plus-diagnosis-fix` `IN-REVIEW` developer - The requested fix was already in the tree; this entry claims it with evidence and closes two gaps. Already done: plugin `CLAUDE.md` "Building" and `CONTRIBUTING.md` step 2 document `-NoHotReloadFromIDE` for hand-run `Build.bat`/`Build.sh`; `editor_build` (`Content/Python/mcp_proxy.py`, argv at the `-WaitMutex", "-NoHotReloadFromIDE"` line) always passes it, pinned by `test_mcp_proxy_editor_start.EditorBuildTest.test_windows_argv_and_immediate_return`; the "target editor must be closed" constraint is enforced, not just documented: `editor_build` refuses with `BUILD_BLOCKED_BY_EDITOR` while any `UnrealEditor%` process of this checkout runs (the census includes `UnrealEditor-Cmd`, so it also covers the LNK1104 holder from #1), pinned by `test_an_editor_of_this_checkout_blocks_the_build_and_is_named`; `docs/wiki-src/mcp-transport.md` -> Building carries the contract. "Never trust Build.bat's exit code" is handled: `editor_build_status` reports `failed` when exit is 0 but UBT's `Result:` line is not `Succeeded`. Gaps closed here: (1) `CLAUDE.md` blamed "another project's `LiveCodingConsole.exe`", which contradicts this ticket and engine source (`Developer/Windows/LiveCoding/Private/LiveCodingModule.cpp:1136-1150` creates `Global\LiveCoding_<exe path>` in the editor process); rewritten to name the other editor, the shared `UnrealEditor.exe`, and that killing the console does not help. (2) No test pinned the exit-0/`Result: Failed` rule or the Live Coding refusal line in `errors`: added `test_mcp_proxy_editor_start.EditorBuildStatusTest.test_live_coding_refusal_with_exit_zero_is_failed`, which fails when `succeeded = exit_code == 0 and (...)` is reduced to `exit_code == 0` (mutation-checked on a scratch copy). Not reproduced: the mutex is Windows-only (LiveCoding module is `Developer/Windows`, Build.cs gates on Win64), so on this Linux box nothing creates it and the cross-project block cannot occur or be exercised; the -NoHotReloadFromIDE gate logic was verified by reading `HotReload.cs:270-317` only. Reviewer on Win64: open any other project's editor on the same engine with Live Coding on, run `editor_build`, expect `Result: Succeeded`. Not done: "back up Binaries/Win64 before building" was not mechanized (the editor-census refusal removes the observed cause). Files: `CLAUDE.md`, `Content/Python/tests/test_mcp_proxy_editor_start.py`.
- `#4-linux-verification` `IN-REVIEW` tester — Fix commit d092fbef. Python run3: 462 tests OK (5 skipped, none of them these). This includes `EditorBuildTest.test_windows_argv_and_immediate_return` (`editor_build` always passes `-NoHotReloadFromIDE`), `.test_an_editor_of_this_checkout_blocks_the_build_and_is_named` (target-editor-closed constraint enforced as `BUILD_BLOCKED_BY_EDITOR`) and `EditorBuildStatusTest.test_live_coding_refusal_with_exit_zero_is_failed` (exit 0 with `Result: Failed` reads `failed`). The doc fix (plugin `CLAUDE.md` / `CONTRIBUTING.md` / `mcp-transport.md` naming the other editor and the shared `UnrealEditor.exe`) is a doc-only item. Remains: the Live Coding mutex is Windows-only, so the cross-project block cannot occur or be exercised on this Linux box, and the fix's actual effect is unverified. Not mechanized: "back up Binaries/Win64 before building". To close: a Win64 reviewer opens another project's editor on the same engine with Live Coding on, runs `editor_build` for this project, and expects `Result: Succeeded`.
