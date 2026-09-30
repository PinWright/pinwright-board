---
id: B-asset-delete-runs-nested-in-validate-on-save-slow-task
title: "`asset.delete` right after `asset.save {force:true}` ran nested inside the editor's post-save data-validation slow task (FlushRenderingCommands pumping the GameThread queue); its own delete progress UI then asserted in Slate and killed the editor"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, delete, save, dispatch, safepoint, reentrancy, render-flush, slow-task, editor-crash]
encounters: 1
lastSeen: 2026-09-29T20:22:10Z
---

# A delete issued right after a batch of saves re-entered the editor's validation modal

Sequence (one Python client, calls strictly sequential, no PIE): seven `asset.save {assetPath, force:true}` on
freshly created materials under `/App/MultiplayerLevelEditor/` (all returned `saveState: written`), then one
`asset.delete {paths:[those seven]}`. The HTTP connection dropped and the editor died:

```
Assertion failed: IsValid() [SharedPointer.h:1070]
  FSlateWindowHelper::FindWindowByPlatformWindow
  FSlateApplication::LocateWindowUnderMouse
  FSlateUser::UpdateTooltip
  FSlateApplication::TickPlatform / Tick
  TickSlate                                        FeedbackContextEditor.cpp:428
  FFeedbackContextEditor::ProgressReported
  FFeedbackContext::RequestUpdateUI
  ObjectTools::DeleteItems                         ObjectTools.cpp:3207
  ObjectTools::DeleteObjects
  AssetDeletePolicy::DeleteAsset                   AssetDeletePolicy.cpp:181
  AutoHandler_318_                                 AssetManageHandler.cpp:1252
  FRpcDispatcher::ProcessRequest                   RpcDispatcher.cpp
  TGraphTask<FAsyncGraphTask>::ExecuteTask
  FNamedTaskThread::ProcessTasksUntilIdle
  FRenderCommandFence::Wait
  FFrameEndSync::Sync / FlushRenderingCommands
  FSlateApplication::AddModalWindow
  FFeedbackContextEditor::StartSlowTask
  FSlowTask::MakeDialog
  UEditorValidatorSubsystem::ValidateAssetsInternal
  UEditorValidatorSubsystem::ValidateOnSave
  UEditorValidatorSubsystem::ValidateAllSavedPackages   (FTimerManager::Tick, UEditorEngine::Tick)
```

The log shows `LogRendererCore: Warning: FlushRenderingCommands called recursively! 2 calls on the stack.`
just before the assert. So the dispatcher's GameThread task for `asset.delete` was drained by
`ProcessTasksUntilIdle` inside the render fence wait that the editor's deferred on-save validation opened
while creating its slow-task modal - the safe-point gate / reentrancy guard did not hold it back. The delete
then opened its own slow task and ticked Slate from inside the half-built validation modal.

Possible contributor (the reporter's doing, noted for honesty): about five minutes earlier the "PinWright
Setup" Slate window had been closed from outside with `wmctrl -c` (it covered a capture region and ignored
`xdotool windowmove`). A stale platform window could explain the invalid shared pointer in
`FindWindowByPlatformWindow`; the nesting above is a defect either way.

Result: the seven `.uasset` files stayed on disk (removed by hand after the crash). No work lost.

Evidence: `Saved/Crashes/crashinfo-PDS-pid-3804922-01A0EED4FC4D770DB7B16D49ADD83EB4`, `Saved/Logs/PDS.log`
(backed up on next start, lines ~6590-6660). Host `/sdb-disk/src/unreal/unreal-fpv` (Linux), UE 5.8, plugin `8fcc0b2a`.

**Fix (proposed):** treat "inside `GIsSlowTask` / a modal slow task / a render-fence wait" as not-safe in the
safe-point gate and defer the request, or have `asset.delete` (and any verb that runs `ObjectTools` with
progress UI) refuse with a retryable code while `ValidateAllSavedPackages` is pending.

## History
- `#1-delete-after-force-saves` `OPEN` reporter - First sighting, as above.
- `#2-defer-inside-slow-task` `IN-REVIEW` developer - Root cause: `FRpcDispatcher::ProcessRequest` (`Dispatch/RpcDispatcher.cpp`) deferred every request only on Saving/GC, and the safe-point gate (`IsTickUnsafeMethod && !IsSafeNow`) covers only the tick-unsafe table, which does not list `asset.delete`. Nothing asked "is a slow task still open on this stack", so the render-fence wait inside the validation's `FSlowTask::MakeDialog` drained and ran the delete. Fix: new `PinWrightSafePoint::IsInsideSlowTask()` (`Dispatch/SafePoint.h`: `GIsSlowTask || GWarn->GetScopeStack().Num() > 0`), added as a third term to the all-methods Saving/GC defer in `ProcessRequest`, so every request drained inside an open slow task (dialog or not) goes to `PendingQueue` and runs after that stack unwinds. The subsystem tick drain re-enters `ProcessRequest` and re-defers while the slow task is still open. Regression: `PinWright.core.safe_point.DispatcherDefersInsideSlowTask` (`Tests/World/TestSafePointGate.cpp`) opens a real `FScopedSlowTask`, drives the ungated `_test.beta` fixture through `ProcessRequest`, and asserts it neither runs inline nor on a drain inside the scope, then runs once the scope ends. Not proven live: the exact validate-on-save repro was not re-run.
