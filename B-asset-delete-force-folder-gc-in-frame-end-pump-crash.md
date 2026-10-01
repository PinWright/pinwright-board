---
id: B-asset-delete-force-folder-gc-in-frame-end-pump-crash
title: "`asset.delete {path: <folder>, force: true}` ran inside the frame-end render-fence pump (FFrameEndSync::Sync -> ProcessTasksUntilIdle); its force delete's GC tore down a world whose audio device pumps the task graph again, tripping the TaskGraph RecursionGuard assert and killing the editor"
status: OPEN
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
