---
id: B-editor-not-ready-blames-benign-work-when-gpu-is-removed
title: "EDITOR_NOT_READY attributes a wedged game thread to benign engine work and tells the caller to wait, with no check for GPU device removal — after DXGI_ERROR_DEVICE_HUNG the thread never drains and the advice is the opposite of correct"
status: DONE
severity: Medium
category: ergonomic
tags: [editor, EDITOR_NOT_READY, watchdog, diagnosis, gpu, device-removed, DXGI_ERROR_DEVICE_HUNG, shared-editor, multi-agent, world-lock]
encounters: 1
costly: 1
lastSeen: 2026-09-08T15:33:00Z
---

# The stall diagnosis names three benign causes and omits the fatal one

Every PinWright RPC issued against a wedged game thread returns `EDITOR_NOT_READY` with a
diagnosis that is well written and, in this case, wrong in the one way that matters:

```
The Unreal editor's game thread has not completed a tick for 269 s. [...] No PinWright RPC is
in flight, so the stall is engine-internal work (asset compile, map load, package save) rather
than a PinWright call. [...] wait if the operation is expected to be this long, otherwise kill
and restart the editor. [...] Retrying is harmless - the condition clears by itself if the
thread drains.
```

Three of those sentences are advice to wait. The enumeration "asset compile, map load, package
save" is presented as the explanation, and all three are recoverable. The actual cause was none
of them.

## Measured (this checkout, 2026-09-08)

The editor's GPU had already been declared removed **before my first call**:

```
[2026.09.08-15.26.07:599] LogD3D12RHI: Error: GPU crash detected:
    - Device 0 Removed: DXGI_ERROR_DEVICE_HUNG
[2026.09.08-15.26.08:630] LogD3D12RHI: Error: PageFault at VA "0xFFFFDF383E3F1000" (GPU 0)
    CrashType: GPUCrash, Aftermath dump D3D12.0.2026.09.08-19.26.07.nv-gpudmp
```

I called `editor.list_dirty_packages` at 15:30, 15:30 and 15:32 and got the same diagnosis with a
rising counter — 269 s, 285 s, 369 s. A counter that only ever rises is itself the signal that
nothing is draining, and the message says the opposite ("the condition clears by itself"). I only
learned the truth by reading `Saved/Logs/EAContentExamples58.log` myself.

The underlying crash is already tracked as
`B-editor-screenshot-pie-gpu-pagefault-during-shader-late-association` (Critical, filed by the AI
stream at 15:26:08Z). **This ticket is not about the crash** — it is about the fact that the next
stream to touch the editor is told to wait for it.

## What it cost

ENV took the world lock at 15:30:10Z for a 15-frame acceptance re-shoot, spent the slot retrying
and waiting on advice that could never come good, and released at 15:33 having captured nothing.
In a shared editor with five queued streams, every one of them pays this separately: the message
gives each of them the same reason to wait.

## What would close it

UE already knows. `GIsGPUCrashed` is set on the device-removed path, and the RHI holds the removal
reason. When the watchdog reports a stalled game thread it should check that state first and, when
set, replace the benign enumeration and the wait advice with the removal reason and an
unambiguous verdict — the editor is dead, restart it, nothing will drain. Cheap, and it converts a
4-minute discovery into a 1-call one.

Two smaller improvements in the same message, independent of the GPU case:

- The stall counter is monotone. Reporting whether it has advanced since the previous observation
  would let a caller distinguish "long operation in progress" from "thread is dead" without any
  GPU-specific knowledge.
- "Retrying is harmless" is true of the call and false of the situation: retrying cost a lock
  slot. Harmless-to-the-server is not harmless-to-the-caller in a queued, single-world editor.

## History
- `#1-filed` `OPEN` ENV reporter — Hit 2026-09-08 15:30-15:33Z on the FPS compound map (map as
  forcing function, host `CLAUDE.md`), UE 5.8. Took the world lock for the build-08 exterior
  re-shoot, and every RPC returned `EDITOR_NOT_READY` blaming "engine-internal work (asset
  compile, map load, package save)" and advising a wait, while the log had recorded
  `Device 0 Removed: DXGI_ERROR_DEVICE_HUNG` at 15:26:07 — four minutes before I took the lock.
  Retried three times across ~4 minutes (counter 269 -> 285 -> 369 s, monotone) before reading the
  log and diagnosing it myself; the slot was lost and released empty. Asked for a `GIsGPUCrashed` /
  device-removal check ahead of the benign enumeration, plus a monotone-counter hint and softening
  of "retrying is harmless". Filed at Medium as an ergonomic/wrong-diagnosis issue rather than a
  crash. On reach: the AI stream's crash ticket records the same `EDITOR_NOT_READY` at 131 s from
  `editor.status`, so a second stream did see this message in the same incident — I have not bumped
  severity on that alone because AI's report treats it as a symptom rather than as something that
  misled them, and I will not claim a cost I did not observe on their side.
- `#2-gpu-crash-verdict` `IN-REVIEW` developer — Still reproducible in source before the fix:
  `GameThreadStalledMessage` (Transport/McpRequestCore.cpp) emitted the benign "asset compile, map
  load, package save" list, the wait advice and "Retrying is harmless - the condition clears by
  itself" with no GPU check; the plugin's code is `EDITOR_GAME_THREAD_STALLED`, and the reporter saw
  `EDITOR_NOT_READY` because the stdio proxy's `_probe_state` mapped every `editorReady:false` ping to
  the retryable `not_ready` state. Fix: the I/O thread reads `GIsGPUCrashed` (CORE_API, set by every
  RHI's device-removed/lost path) when it reports a stall (`FRequestConfig::bGpuCrashed`,
  SocketHttpServer.cpp). With it set, `ping` adds `gpuCrashed: true`, `retryable: false`, the
  message names the GPU crash and gives the restart verdict (no benign list, no wait advice) and
  points at the log for the removal reason, and `tools/call` is refused immediately instead of
  queueing behind a dead thread. The proxy maps `gpuCrashed:true` to `unresponsive`
  (`EDITOR_UNRESPONSIVE`, retryable:false, the editor's message as text). Independent of GPU: the
  plain-stall message now explains that a larger `stalledSeconds` on a later call means no tick
  completed in between (thread not draining), drops "clears by itself", and says retrying does not
  shorten the stall and a world lock should not be held while polling. The RHI's textual removal
  reason is not exposed through any cross-RHI API, so the message points at the log rather than
  quoting it. Files: Source/PinWright/Private/Transport/{McpRequestCore.h,McpRequestCore.cpp,
  SocketHttpServer.cpp}, Source/PinWright/Private/Tests/Transport/TestGameThreadStallProbe.cpp,
  Content/Python/mcp_proxy.py, Content/Python/tests/test_mcp_proxy_editor_start.py,
  docs/wiki-src/{unattended,mcp-transport,system}.md, CHANGELOG.md. Tests:
  `PinWright.transport.liveness.Ping.StalledOnCrashedGpuIsTerminal`,
  `PinWright.transport.liveness.ToolsCall.CrashedGpuStallGatesDispatch`, new assertions in
  `PinWright.transport.liveness.Ping.StalledWithoutInFlightOmitsMethod`; Python
  `tests.test_mcp_proxy_editor_start.GpuCrashedStallProbeTest` (2 tests, pass).
- `#3-verified-linux` `DONE` tester — Fix commit 40ff81c8. Passed non-skipped in run3/full: `PinWright.transport.liveness.Ping.StalledOnCrashedGpuIsTerminal`, `PinWright.transport.liveness.ToolsCall.CrashedGpuStallGatesDispatch` and `PinWright.transport.liveness.Ping.StalledWithoutInFlightOmitsMethod`, plus the rest of `PinWright.transport.liveness` (all pass). Python run3 OK includes `GpuCrashedStallProbeTest.test_gpu_crashed_stall_is_terminal_not_retryable` and `.test_plain_stall_stays_retryable_not_ready`. Acceptance: with the GPU crashed, the ping reports `gpuCrashed:true`, `retryable:false` and a restart verdict with no benign list and no wait advice; `tools/call` is refused at once; the proxy maps it to `EDITOR_UNRESPONSIVE`. The plain stall message explains that a growing `stalledSeconds` means nothing drained, drops "clears by itself", and says retrying does not shorten the stall and not to hold a world lock. Wiring: `SocketHttpServer.cpp:972` sets `bGpuCrashed = GIsGPUCrashed` (source read). Coverage limits: the tests inject `bGpuCrashed`, and no real D3D12/Vulkan device removal was produced. The removal reason is pointed at in the log rather than quoted, since there is no cross-RHI API for it. The counter delta is explained in text, not reported as a field.
