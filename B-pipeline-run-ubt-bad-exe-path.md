---
id: B-pipeline-run-ubt-bad-exe-path
title: "pipeline.run_ubt always fails CREATEPROC_FAILED — hardcoded RunUBT.bat does not exist on Windows"
status: IN-REVIEW
severity: High
category: bug
tags: [pipeline, ubt, createproc, windows, dead-path, build]
---

# pipeline.run_ubt always fails CREATEPROC_FAILED — hardcoded RunUBT.bat does not exist on Windows

`pipeline.run_ubt` is non-functional on Windows: it can **never** launch Unreal
Build Tool regardless of valid arguments. Every invocation returns
`status=running` synchronously, then the async job terminates in ~1–2 ms with
`error="CREATEPROC_FAILED"` and an empty `progress[]` — UBT never spawns.

Root cause (`Handlers/Build/PipelineHandler.cpp:35-36`): the handler hardcodes
the UBT entry point as

```cpp
const FString UbtExe = FPaths::ConvertRelativePathToFull(
    FPaths::EngineDir() / TEXT("Build/BatchFiles/RunUBT.bat"));
```

There is **no `RunUBT.bat`** in the Windows engine layout — only `RunUBT.sh`
(Mac/Linux). The Windows `Engine/Build/BatchFiles/` ships `Build.bat`,
`Rebuild.bat`, `RunUAT.bat`, etc., but no `RunUBT.bat`. `CreateProc` is then
handed a path to a file that does not exist, returns an invalid handle, and
`ProcPollBind.h:22-25` reports `CREATEPROC_FAILED`. (Even if the file existed,
`CreateProc` cannot launch a bare `.bat` as the application image on Windows
without a shell — but here the file is simply absent, so the call dies before
that ever matters.)

This is distinct from the two DONE tickets
`B-pipeline-run-ubt-no-completion-signal` /
`B-system-run-ubt-no-completion-signal`, which fixed the *missing completion
signal* (the job ticker now reports the terminal state correctly — that part
works). Both of those tickets explicitly **SKIPPED** the live UBT launch
(`#3-skip-launches-ubt`), so the dead executable path was never exercised. The
completion machinery now faithfully reports a failure that the handler
guarantees on every Windows call.

Secondary symptom (same root cause, do not file separately): because UBT never
launches, `platform="NotARealPlatform"` produces the **byte-identical**
`CREATEPROC_FAILED` as a valid `Win64/Development` build — there is no input
validation and no way for a CI caller to distinguish a typo from a real build
failure by error shape. Fixing the exe path (so real builds run and bad
platforms get UBT's own validation error) resolves both.

**Verbatim repro (replayed live via mcp__editor-automation__call):**

Valid build:
- `pipeline.run_ubt` `{target:"UnrealEditor", platform:"Win64", configuration:"Development"}`
  → `{"status":"running","ticket_id":"j_20260621T120006_c19f99c4",...,"args":"UnrealEditor Win64 Development"}`
- `system.job_status` `{ticket_id:"j_20260621T120006_c19f99c4"}`
  → `{"status":"failed","started_at":"...12:00:06.670Z","completed_at":"...12:00:06.671Z","error":"CREATEPROC_FAILED","progress":[]}`

Bogus platform (identical terminal error):
- `pipeline.run_ubt` `{target:"UnrealEditor", platform:"NotARealPlatform", configuration:"Development"}`
  → `{"status":"running","ticket_id":"j_20260621T120009_bd5db86c",...,"args":"UnrealEditor NotARealPlatform Development"}`
- `system.job_status` `{ticket_id:"j_20260621T120009_bd5db86c"}`
  → `{"status":"failed",...,"error":"CREATEPROC_FAILED","progress":[]}`

**Impact:** the entire `pipeline.run_ubt` capability is dead on Windows — a CI
smoke-build, the most natural use of this RPC, cannot run. The synchronous
`status=running` is a false-success signal; only a follow-up `system.job_status`
reveals the guaranteed failure.

**Fix:** Point the handler at the real Windows UBT entry point. Use
`Build/BatchFiles/Build.bat` (the cross-platform forwarding wrapper) launched via
the platform shell, or invoke
`Binaries/DotNET/UnrealBuildTool/UnrealBuildTool.exe` directly (a real
executable image `CreateProc` can launch). Select the entry point per platform
(`RunUBT.sh` on Mac/Linux, the `.exe` or `Build.bat`-via-`cmd /c` on Windows)
rather than hardcoding a single `.bat` name. Once UBT actually runs, an invalid
platform/configuration surfaces UBT's own diagnostic instead of the generic
`CREATEPROC_FAILED`, also resolving the typo-indistinguishability symptom.

## History
- `#1-initial-repro` `OPEN` reporter — `pipeline.run_ubt` returns `status=running` then fails `CREATEPROC_FAILED` in ~1–2 ms for every input. Root cause: handler hardcodes `Engine/Build/BatchFiles/RunUBT.bat` (`PipelineHandler.cpp:35`), which does not exist on Windows (only `RunUBT.sh` ships). Confirmed by replaying valid `UnrealEditor/Win64/Development` and bogus `NotARealPlatform` invocations — both `CREATEPROC_FAILED` — and by `ls Engine/Build/BatchFiles/` showing no `RunUBT.bat`. Distinct from the DONE no-completion-signal tickets, which SKIPPED the live launch and never hit this dead path.
- `#2-fix-mirror-system-run-ubt` `IN-REVIEW` developer — Replaced the hardcoded `Build/BatchFiles/RunUBT.bat` with per-platform resolution of the real forwarding wrapper (`Build.bat` on Windows / `Build.sh` elsewhere), mirroring the proven sibling `system.run_ubt` (`SystemControlHandler.cpp:360-370`), plus a `FPaths::FileExists` guard returning a distinct `UBT_NOT_FOUND` instead of generic `CREATEPROC_FAILED`. Extracted the selection into a shared free function `EARG_Ubt::ResolveUbtEntryPoint()` (`Handlers/Build/UbtEntryPoint.h`) so the handler and its regression test exercise identical production code. Empirically settled the adversarial REWORD's premise: `CreateProcess(NULL, "\"…\\Build.bat\" …")` returns OK on this host (Win32 resolves a `.bat` via the command processor), so `Build.bat` is a working image and no `cmd /c` / `UnrealBuildTool.exe` rework is needed — and the sibling `system.run_ubt` (also `Build.bat`) is NOT a second dead bug, so no rescope. Ticket text already named `Build.bat`/platform-branch as the fix, so no reword. Files: `Private/Handlers/Build/PipelineHandler.cpp`, new `Private/Handlers/Build/UbtEntryPoint.h`. Test: added `FBuildRunUbtEntryPointExistsTest` (`pipeline.run_ubt.EntryPointResolvesToExistingScript`) in `Private/Tests/EditorOps/TestBuildHandlers.cpp` — asserts the resolver never selects `RunUBT.bat`, selects `Build.bat`/`Build.sh` per platform, and that the resolved script exists on disk (fails under the reverted dead path); never spawns a build. Not yet compiled/tested (later phase).
