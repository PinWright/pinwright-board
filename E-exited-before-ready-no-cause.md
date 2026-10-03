---
id: E-exited-before-ready-no-cause
title: "editor_restart / editor_start EDITOR_EXITED_BEFORE_READY gives only the exit code: no log path and no fatal line, and the crash log may be PDS_2.log rather than the usual PDS.log"
status: OPEN
severity: Low
category: ergonomic
tags: [editor_start, editor_restart, startup, crash, oom, vulkan, log-path, error-message]
encounters: 1
lastSeen: 2026-09-30T12:16:12Z
rice: [2, 2, 1, 1]
priority: 33
---

# A startup death is reported without its cause or its log

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `adb239fd`, shared GPU (another
checkout's editor held 5.8 of 8 GB of VRAM). `editor_restart {mode:"visible", discard:true, extra_args:[...]}`
quit the running editor and then answered:

```
EDITOR_EXITED_BEFORE_READY: editor (pid 976019) exited with code 1 before answering RPCs.
Command: ... PDS.uproject ...
```

That was all the information given. The cause was a Vulkan out-of-memory crash on the render thread at
frame 0 (`VulkanRHI::FMemoryManager::AllocateBufferMemory` ... `FRDGBuilder::AllocateTransientResources`,
then `FUnixPlatformMisc::RequestExit(1, GenericPlatformmallocCrash::Malloc.OutOfMemory)`). It was not in
`Saved/Logs/PDS.log`. The new editor started while the old one still held that file, so it wrote
`Saved/Logs/PDS_2.log` instead. Finding it took an `ls -t` of the log directory and reading the tail. The
fix was to relaunch with `-dpcvars=r.Streaming.PoolSize=200,r.RDG.TransientAllocator=0`.

**Expected:** `EDITOR_EXITED_BEFORE_READY` (and `EDITOR_EXITED_BEFORE_TESTS`) carries the log file the
exited PID actually wrote (it can be found by PID or by the `Log file open` time), plus the last
`Fatal`/`Error`/`RequestExit` line, as `EDITOR_STARTUP_CEF_RACE` already does with `cefLine`. A GPU OOM
on a shared box is common enough that a named `EDITOR_STARTUP_OOM` would also help.

## History
- `#1-filed-vram-oom-restart` `OPEN` reporter - Filed during the PDS QA #830 repro (wt1). The restart died of Vulkan OOM at frame 0 and the error named only exit code 1. The crash log was `PDS_2.log`, because the previous instance still held `PDS.log`. Cost: a directory listing and a log read before the relaunch with low-VRAM cvars.
