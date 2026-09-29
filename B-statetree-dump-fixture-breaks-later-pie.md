---
id: B-statetree-dump-fixture-breaks-later-pie
title: "Leaked test fixtures (schema-less StateTrees, an open Level Sequencer) fire engine ensures in the next owned-PIE test; the ensure stall times out drive.observe's PIE-stop wait"
status: IN-REVIEW
severity: Medium
category: bug
tags: [tests, pie, fixture-leak, statetree, sequencer, ensure, gap-analysis-2026-09-28]
encounters: 1
costly: 1
lastSeen: 2026-09-29T10:22:00Z
---

# Leaked fixtures break the next owned-PIE test

`PinWright.drive.observe.ScreenshotIncludesUmgAndMarksAtSurfaceLocalCoords` failed on the full
offscreen suite (`Saved/Logs/pw_gapwave_full_offscreen2.log:26626-27017`) with
`Timed out waiting for the owned PIE session to stop.` Its capture assertions all passed. Three
defects combine; none is in the victim test's subject under test.

**1. Schema-less StateTree fixtures survive forever (root cause of the PIE-start ensure).**
`Tests/Assets/TestStateTreeDumpBuilder.cpp` (`PinWright.Assets.StateTree.DumpBuilder.Shape` and
`...AssetDump.WritesStateTreeAspectFile`) author `/Engine/Transient/ST_StateTreeDump_<guid>` with
`RF_Public | RF_Standalone | RF_Transient`, no schema, and tear down with only `RemoveFromRoot()`.
`RF_Standalone` keeps both trees alive through every suite GC; there is no transient-package
detach at all. Authoring them (`AddRootState` / `AddChildState` -> `UStateTreeEditingSubsystem::MarkAsModified`)
put them on the StateTree compiler manager's dirty list (`DirtyStateTrees_AnyThread`, a
`TSet<TObjectKey<UStateTree>>`, weak). At the next PIE start `FCompilerManagerImpl::HandlePreBeginPIE`
(`StateTreeCompilerManager.cpp:773-799`) resolves the still-live keys, `QueueForCompilation` ->
`AllowQueuedCompilation` hits `ensure(EditorData->Schema)` (`:469`), then logs
`The state tree '...ST_StateTreeDump_...' does not have a schema.` for both trees. The ensure
report stalled the game thread 33.0 s (`SendNewReport - 31.731 s`); PIE start took 39.7 s.
`editor.simulate_input.PiePlayerInputDelivery` already skips on the same leaked tree
(`reason=pie-start-triggers-engine-ensure`, log line 28226).

**2. The PIE-stop timeout is a consequence of #1 plus a mis-anchored deadline.**
`TestDriveObserveScreenshotComposite.cpp` set `CleanupDeadline = RunTest + 35 s`. The ensure-stalled
PIE start alone took ~40 s, so the first cleanup poll after `FEndPlayMapCommand` was already past the
deadline and failed while EndPlayMap was still in flight. The stop budget must be measured from the
stop request.

**3. Separate defect: an open Level Sequencer left by camera.animation_shots tests.**
The EndPlayMap ensure `UE::GetPlayInEditorID() == INDEX_NONE` (`UnrealEngine.cpp:767`, 7.4 s stall)
is not caused by #1. Stack: `FGameDelegates::EndPlayMapDelegate` (`PlayLevel.cpp:425`, after the PIE
world contexts are removed) -> `FLevelEditorSequencerIntegration::OnEndPlayMap` -> `FSequencer::ForceEvaluate`
-> `FActorLocatorFragment::Resolve` -> `FTemporaryPlayInEditorIDOverride(0)` with no PIE world context
left. It fires only when a level Sequencer is open during PIE end. The open one was
`MCP_AnimShotsSubjSeq_52F69...`, opened by `camera.animation_shots.BurstResponseShapeSurvives`
(log 23605; the verb reports `opened` and leaves Sequencer open by design) and never closed: the
fixture `FPWAnimSubjMovingFixture` (`Tests/Render/TestAnimationShotsSubjects.cpp`) detached the
sequence asset but left its editor open. Engine-side behavior; the plugin-side fix is closing the
editor the fixture caused to open.

Side finding, not fixed here: the same fixture (and the other `sequencer.*` probes) create and save
assets under `/Game/MCP_*Probe/`, outside the `/Game/PinWrightTests` scratch root; the host's
`Content/` carries 20 empty `MCP_*Probe` directories left by suite runs.

**Fix:** see History.

## History
- `#1-full-suite-pie-timeout` `OPEN` reporter — Filed from the full offscreen suite (`pw_gapwave_full_offscreen2.log`): drive.observe composite screenshot failed on the PIE-stop wait after two engine handled ensures (StateTree schema at PIE start, Sequencer PIE-ID at EndPlayMap). Root-caused to the two leaked fixtures above plus the RunTest-anchored cleanup deadline. Cost: one full-suite verdict carried a false failure.
- `#2-fixture-teardown-and-stop-budget` `IN-REVIEW` developer — (a) `Tests/Assets/TestStateTreeDumpBuilder.cpp`: every exit path now calls `PWStateTreeDumpDiscardFixture` = `PwTestAssetTeardown::DiscardLoadedAssetNoGc` (clears `RF_Standalone`, detaches into the transient package) + `MarkAsGarbage()`, so the compiler manager's weak dirty-list key and the simulate_input schema probe (`IsValid`) stop seeing the tree immediately, with no GC dependency. (b) `Tests/Render/TestAnimationShotsSubjects.cpp`: `~FPWAnimSubjMovingFixture` closes the sequence's asset editors (`CloseAllEditorsForAsset`, same as `TestSequencerBakeControlRig`) before `CleanupTestAsset`. (c) `Tests/Drive/TestDriveObserveScreenshotComposite.cpp`: the 15 s PIE-stop budget starts at the first cleanup poll instead of RunTest+35 s. No skip added. `-SingleFile` compile of all three: clean. Not fixed: `editor.simulate_input.PiePlayerInputDelivery` has the same RunTest-anchored cleanup deadline; the `/Game/MCP_*Probe` scratch-root violation.
- `#3-sibling-deadline-and-scratch-root` `IN-REVIEW` developer — Closed the two items left open in #2. (a) `Tests/Drive/TestDriveGameInput.cpp` (`editor.simulate_input.PiePlayerInputDelivery`): same fix, the PIE-stop budget starts at the first cleanup poll. (b) The 20 fixtures that saved under `/Game/MCP_*Probe` (`Tests/Render/TestAnimationShotsSubjects.cpp`, `Tests/Render/TestAnimationCaptureHandlers.cpp`, 18 `Tests/Sequencer/*.cpp`) now build `DestFolder` as `PinWrightSuiteMaintenance::ScratchRootPackagePath() / "MCP_<X>Probe"`; the two `TestSequencerBakeControlRig` export paths moved the same way. The zz_suite_end sweep now owns them. Unchanged: never-written "absent"/"DoesNotMatter" path literals. The 20 empty, untracked `Content/MCP_*Probe` directories in the host were removed (0 files, `git ls-files Content/MCP_*` empty); the full suite running on the older binary may recreate some. `-SingleFile` compile of all 21 files: clean.
