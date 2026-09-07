---
id: B-live-coding-mutex-blocks-cross-project-build
title: "Building the plugin fails while ANY other UE project is open, because the Live Coding mutex is keyed on the shared UnrealEditor.exe path"
status: OPEN
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
