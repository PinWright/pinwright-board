---
id: B-asset-delete-force-folder-gc-in-frame-end-pump-crash
title: "`asset.delete {path: <folder>, force: true}` ran inside the frame-end render-fence pump (FFrameEndSync::Sync -> ProcessTasksUntilIdle); its force delete's GC tore down a world whose audio device pumps the task graph again, tripping the TaskGraph RecursionGuard assert and killing the editor"
status: DONE
severity: High
category: bug
tags: [asset, delete, force-delete, dispatch, safepoint, reentrancy, gc, frame-end-sync, editor-crash]
encounters: 1
lastSeen: 2026-10-01T13:29:30Z
---

# A force folder delete, dispatched from the frame-end task pump, crashed the editor in GC

Offscreen editor (`-RenderOffScreen -unattended`, Linux, UE 5.8, plugin `0a3b3acc`), no PIE. Eleven scratch
assets under `/Game/PinWrightTests/LayoutVision` (8 Blueprints, a material, an anim BP, a Control Rig), each
opened, screenshotted and closed with `editor.close_asset`. `asset.delete {path: folder}` was refused with
`ASSET_IN_USE` (only self-references plus the undo buffer). Retried with `force: true`: the stream dropped and
the editor died.

```
Assertion failed: ++Queue(QueueIndex).RecursionGuard == 1   TaskGraph.cpp:705
  FNamedTaskThread::ProcessTasksUntilIdle
  FAudioDevice::Teardown                          AudioDevice.cpp:917
  FAudioDeviceManager::DecrementDevice
  UWorld::SetAudioDevice / UWorld::BeginDestroy   (an asset editor preview world being GC'd)
  UnhashUnreachableObjects / CollectGarbage
  UPackageTools::UnloadPackages
  ObjectTools::CleanupAfterSuccessfulDelete
  ObjectTools::ForceDeleteObjects
  UEditorAssetSubsystem::DeleteDirectory
  AssetDeletePolicy::DeleteDirectory              AssetDeletePolicy.cpp:226
  AutoHandler (asset.delete)                      AssetManageHandler.cpp:1251
  FRpcDispatcher::ProcessRequest
  TGraphTask<FAsyncGraphTask>::ExecuteTask
  FNamedTaskThread::ProcessTasksUntilIdle         TaskGraph.cpp:706
  FFrameEndSync::Sync                             RenderingThread.cpp:2565
  FEngineLoop::Tick
```

The dispatcher's GameThread task was drained by `ProcessTasksUntilIdle` inside `FFrameEndSync::Sync`. The
delete's GC then destroyed a preview world (one of the just-closed asset editors' worlds), whose audio device
teardown pumps the GameThread queue again: a nested `ProcessTasksUntilIdle` on the same named thread, which
the engine asserts against. Same family as `B-asset-delete-runs-nested-in-validate-on-save-slow-task` (that
fix defers only while a slow task is open; here no slow task was open) and `B-python-execute-reentrant-gc-crash`
(GC from inside a nested pump).

**Fix direction:** a request whose work can run `CollectGarbage` (force delete, folder delete, unload) must
not execute while the GameThread queue is being drained from inside another pump (`FFrameEndSync::Sync`,
render-fence waits); defer it to the safe point at the top of the next tick, e.g. by listing `asset.delete`
in the tick-unsafe table or by treating any nested named-thread pump as not-safe.

Result: nine of the eleven `.uasset` files stayed on disk and were removed by hand. Evidence:
`Saved/Crashes/crashinfo-PDS-pid-2948183-01A0F7A7E3B57EE094009B4DFD8F67D9` and `Saved/Logs/PDS.log` lines
~4540-4600 in `/sdb-disk/src/unreal/unreal-fpv-wt2` (log copy kept with the vision-check artifacts of
F-graph-layout-core).

## History
- `#1-initial-report` `OPEN` reporter — Hit during the F-graph-layout-core vision check: `asset.delete` with `force:true` on a scratch folder, dispatched from the frame-end task pump, crashed the editor in GC (TaskGraph RecursionGuard assert via audio-device teardown of a preview world).
- `#2-gc-verbs-tick-gated` `IN-REVIEW` developer — Root cause: `IsSafeNow()` already reports unsafe inside `FFrameEndSync::Sync`'s `ProcessThreadUntilIdle(GameThread)` (its `IsInsideNamedThreadPump()` half reads the queue's RecursionGuard), but `FRpcDispatcher::ProcessRequest` only applies that gate to methods in the tick-unsafe table, and no delete verb was listed. The slow-task defer from `B-asset-delete-runs-nested-in-validate-on-save-slow-task` does not fire here because no slow task was open. Fix: listed every remaining verb that runs a synchronous `CollectGarbage` on the handler stack in the tick-unsafe table (`Dispatch/SafePoint.cpp`, family A): `asset.delete`, `asset.bulk_delete`, `level.delete` (all `ObjectTools` delete -> `CleanupAfterSuccessfulDelete` -> `UnloadPackages` -> GC, forced or not) and `source_control.revert` (`ApplyOperationAndReloadPackages`, same GC as the already-listed `asset.reload`). None has a `DispatchMethod` cross-dispatch caller, so the table covers them all; a request drained from the frame-end pump goes to `PendingQueue` and runs from the core-ticker drain at most one 0.1 s pass later, response unchanged. Other GC-reaching verbs were already listed (`asset.reload`, `asset.fixup_redirectors`, `animation.cleanup`, `animation.setup_retargeting`, `animation.retarget_animations`, `level.create`, ...); `GEditor->ForceGarbageCollection` sites only raise a flag and stay ungated. Tests: `PinWright.core.safe_point.AssetDeleteDefersFromNamedThreadPump` (`Tests/World/TestSafePointGate.cpp`) drives the real `asset.delete` (nonexistent path, `force:true`) through `ProcessRequest` from inside `ProcessThreadUntilIdle(GameThread)` — the exact frame-end call — and asserts no inline response, then that `ProcessPendingRequests` answers it; fails if `asset.delete` leaves the table. `PinWright.core.safe_point.KnownVictimsAreGated` now pins all four names. Docs: `docs/wiki-src/asset.md` (asset.delete). Not proven live: the force-folder repro with open-then-closed asset editors was not re-run.
- `#3-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.core.safe_point.AssetDeleteDefersFromNamedThreadPump` passed in w23-final with no skip marker, so the request really ran with the game thread's named-thread pump flag raised (the test skips otherwise): `asset.delete {force:true}` issued from inside `ProcessThreadUntilIdle(GameThread)`, the FFrameEndSync call, produced no inline response and was answered by `ProcessPendingRequests` at the safe point. `KnownVictimsAreGated` (pins asset.delete, asset.bulk_delete, level.delete, source_control.revert) and the other `core.safe_point.*` tests passed. The fix direction (GC-reaching verbs drained from a nested pump defer to the next safe point) is met. Limits: the original force folder delete with closed asset-editor preview worlds was not re-run live, and the test's path does not exist, so no GC ran inside it. Sibling tests `DispatcherDefersFromNestedNamedThreadPump` and `NestedNamedThreadPumpIsUnsafe` skipped `nested-named-thread-pump-not-entered`; they are not this ticket's tests.
