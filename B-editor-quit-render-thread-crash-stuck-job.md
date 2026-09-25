---
id: B-editor-quit-render-thread-crash-stuck-job
title: "editor.quit returned success, then the editor crashed on exit in the render thread (EXCEPTION_ACCESS_VIOLATION in RenderCore ExecuteCommand) with a never-terminal system.run_tests job still registered"
status: IN-REVIEW
severity: High
category: bug
tags: [editor-quit, shutdown, crash, access-violation, render-thread, run_tests, jobs, unclean-exit]
encounters: 1
lastSeen: 2026-09-24T01:16:55Z
---

# editor.quit: render-thread access violation during exit

UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Editor PID 47052 had been up ~4 h with
one `system.run_tests` filter job (`j_20260923T210830_4e5912df`) stuck `running` after its queue
drained (`B-run-tests-filter-job-never-terminal`). No PIE, no asset editors, 0 dirty packages.
`editor.quit {force: true}` answered `requested: true, pieWasActive: false, assetEditorsClosed: 0`.
The log then shows `Engine exit requested (reason: ...)`, `Recording EnginePreExit events`, and the
process died with a crash report (`Saved/Crashes/UECC-Windows-03C02152414870BC3E6EF8B4F8D57331_0000`):
`Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0xffffffffffffffff`, stack entirely
on the rendering thread:

```
RenderCore!ExecuteCommand_NoMarker()           RenderingThread.cpp:1525
RenderCore!ExecuteCommand()                    RenderingThread.cpp:1533
RenderCore!FCommandList::ConsumeCommands<...>  RenderingThread.h:335
RenderCore!FRenderThreadCommandPipe::EnqueueAndLaunch lambda  RenderingThread.cpp:1928
Core!FNamedTaskThread::ProcessTasksNamedThread
RenderCore!RenderingThreadMain
```

A render command whose captured state was already freed ran during teardown. Attribution to
PinWright is not proven: the only unusual state was the stuck automation job and whatever it keeps
alive (the job ticker kept emitting progress every 10 s). Two later `editor.quit` calls on fresh
editors (no stuck job) exited cleanly with no crash report, which is consistent with, but does not
prove, the stuck job being involved. Not a duplicate of `B-editor-quit-crash-pie-active` or
`B-editor-quit-crash-open-asset-editors` (different stack, neither condition present).

Impact: the quit reports success, so a caller cannot tell the exit was a crash without reading
`Saved/Crashes`; the next boot may offer package recovery.

**Workaround:** none needed for the caller beyond checking `Saved/Crashes` after a quit.
**Fix (proposed):** on quit, terminate or detach in-process automation jobs (and their controller
delegates) before requesting exit; report `crashReportAfterExit` is not possible, so at least log
the live job tickets at quit time.

## History
- `#1-render-thread-av-after-quit` `OPEN` reporter — Filed with the stack above; minidump in the
  crash folder. Cheap (no work lost; the editor was being restarted anyway).
- `#2-uaf-and-quit-terminates-jobs` `IN-REVIEW` developer — Attribution, code-traced (not reproduced:
  the Linux host exits cleanly either way, and the minidump is on the Windows host). A job stuck this
  way is not an idle leftover: it is the residue of a heap use-after-free. When `OnTestsComplete`
  fired, `FRunAutomationTestsByFilterJob::Finish` removed its own binding mid-broadcast, which
  destroyed the lambda holding the last ref to the job, and `Finish` then wrote into the freed block
  (`TestsCompleteHandle.Reset()`, the `MoveTemp(OnComplete)` source reset) - see
  `B-run-tests-filter-job-never-terminal#2`. A write into freed heap corrupts whatever the allocator
  hands that block to next, which fits a later fault in an unrelated consumer such as a render
  command reading a `0xffffffffffffffff` pointer; the 4 h gap between the write and the exit fault is
  consistent with that. That use-after-free is fixed at the source (stack keep-alive). Quit is also
  made safe with jobs outstanding: `editor.quit` now ends every job still `running` before it acks
  (`EditorQuitPolicy::TerminateRunningJobs`, `EditorQuitHandler.cpp`) - cancelled through its cancel
  hook when it has one (an isolated `run_tests` child tree is killed), otherwise failed with the new
  `EDITOR_EXITING` code - logs each as a `LogPinWrightSubsystem` warning, and lists them as
  `jobsTerminated`, so a streaming client gets its terminal event before the process goes. Verified
  live on a binary with the old run_tests code: a stuck zero-match job was reported in
  `jobsTerminated`, `jobs.jsonl` recorded `failed/EDITOR_EXITING`, and the editor reached
  `LogExit: Exiting.` with no crash report. Test: `PinWright.editor.quit.TerminatesRunningJobs`.
  Needs a Windows re-check: 57-test filter run, let it drain, `editor.quit`, confirm no report in
  `Saved/Crashes`. Plugin commit `f39443c6`.

