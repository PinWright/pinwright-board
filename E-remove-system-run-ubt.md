---
id: E-remove-system-run-ubt
title: "Remove system.run_ubt: it cannot build the editor target it runs in; agents build from their own shell"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [system, ubt, build, removal, live-coding, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T00:00:00Z
---

# Remove system.run_ubt

`system.run_ubt` spawned UBT from inside the editor. It could not build the editor target of the
editor serving the call (the link needs that editor's DLLs unloaded, and UBT refuses while Live
Coding is active), it built an invalid command line for both call shapes, and its progress watched
a log installed engines never write. Agents build from their own shell, so the verb is deleted
outright per task 11 of plan `Plugins/PinWright/tasks/2026-09-28-gap-quick-wins.md`: no shim, no
alias, no deprecation stub.

**Fix:** deleted the handler and registration in
`Source/PinWright/Private/Handlers/System/SystemControlHandler.cpp`, plus
`Handlers/BuildTools/ProcPollBind.h` and `Handlers/BuildTools/UbtEntryPoint.h` (used only by this
verb; `PipelineHandler.cpp` kept). `ERR_UBT_NOT_FOUND` removed from `Handlers/ErrorCodes.h` (no other
emitter). Its tests removed or repointed. Docs point to a shell build with the editor closed
(`Build.bat`/`Build.sh -Project=... -TargetType=Editor <Platform> Development`) or to
`system.live_coding_compile` for a running editor. `CHANGELOG.md` Unreleased: "Removed
`system.run_ubt`".

**Verifier acceptance criteria:**
1. On a running editor, `call("system.run_ubt", {})` returns method-not-found, and the regenerated
   wiki has no `system.run_ubt` page.
2. `git grep -n -i run_ubt` in the plugin repo shows only the `CHANGELOG.md` entry.
3. `Source/PinWright/Private/Handlers/BuildTools/ProcPollBind.h` and `UbtEntryPoint.h` do not exist.
4. `docs/wiki-src/system.md` ("Building C++" under Cross-cluster overlap),
   `docs/wiki-src/mcp-transport.md`, `docs/arch.md` and the `LIVE_CODING_NOT_AVAILABLE` message in
   `Handlers/System/LiveCodingHandler.cpp` point to a shell build or `system.live_coding_compile`.

**Related:** closed as WONTFIX by this removal: `B-system-run-ubt-missing-target-project`,
`B-system-run-ubt-progress-wrong-log`, `B-system-run-ubt-live-coding-unexplained`.

## History
- `#1-remove-run-ubt` `OPEN` reporter - Filed for task 11 of plan `Plugins/PinWright/tasks/2026-09-28-gap-quick-wins.md` (2026-09-28 gap analysis): the user chose to delete `system.run_ubt` instead of fixing its three open defects, because the verb cannot build the editor target it runs in and agents build from their own shell.
- `#2-deleted-verb-and-helpers` `IN-REVIEW` developer - Deleted the `system.run_ubt` handler, registration and BuildTools includes in `SystemControlHandler.cpp`; deleted `ProcPollBind.h` and `UbtEntryPoint.h`; removed `ERR_UBT_NOT_FOUND` from `ErrorCodes.h` and `docs/error-code-catalog.md`; removed the run_ubt tests in `TestBuildHandlers.cpp`/`TestSystemHandlers.cpp` and repointed references in `TestEditorQuitPolicy.cpp`, `TestSystemInspectMethodsIndexDocs.cpp` and comments; updated `system.md`, `mcp-transport.md`, `arch.md`, `LiveCodingHandler.cpp`, `README.md` op list, `CHANGELOG.md`. Grep gate passes (CHANGELOG only). Not compiled or run yet; verify per the acceptance criteria above.
