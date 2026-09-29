---
id: B-suppress-log-errors-static-leaks
title: "bSuppressLogErrors set by one PinWright test stays on for the rest of the run, so every full-suite green hid engine log errors in all later tests"
status: IN-REVIEW
severity: High
category: bug
tags: [tests, automation, log-suppression, false-green, suite-maintenance, gap-analysis-2026-09-28]
encounters: 1
costly: 1
lastSeen: 2026-09-29T13:10:00Z
---

# Test log-error suppression leaks across the whole suite

`FAutomationTestBase::bSuppressLogErrors`, `bSuppressLogWarnings` and `bElevateLogWarningsToErrors`
are STATIC members (`Core/Private/Misc/AutomationTest.cpp:179-181`). About 60 PinWright sites in 19
files set `bSuppressLogErrors = true` inside `RunTest` (and `TestMaterialHandlers.cpp` sets
`bSuppressLogWarnings`). The engine calls `LoadDefaultLogSettings` (`AutomationTest.cpp:2051`)
whenever the logging test changes, but that only overwrites the flags from
`[/Script/AutomationController.AutomationControllerSettings]` keys that exist. The host defines
`bSuppressLogWarnings` and `bElevateLogWarningsToErrors` but not `bSuppressLogErrors`, so after the
first setter every later test's engine `Error` lines were dropped from its result. Every full-suite
green so far is therefore unverified for engine log errors after the first setter.

Found by the verifier on a filtered run where the setter did not precede them
(`Saved/PinWright/test-runs/e0b2281399ce433aa14c6cfc55d50b49/automation.log`). Five tests failed on
errors a full run had hidden:
`Assets.AnimSequence.AssetDump.WritesAnimSequenceAspectFile`, `Assets.AnimSequence.DumpBuilder.Shape`,
`Assets.AssetResolution.ActorGetComponentsReadsTransientAssetInPie`,
`Assets.AddMetaSoundVariableIntAlias` and `Assets.RemoveMetaSoundNodeAndDisconnect`.

**Fix:** see History.

**Predicted to surface in the next full run.** These are statically derived from
`Saved/Logs/pw_gapwave_full_offscreen2.log`: tests that passed while their window logged
non-controller `Error:` lines, and that neither set suppression nor declare an expectation
themselves. They are not fixed yet.
- Geometry: `Geometry.Ops.Modeling.UVOpsRejectAnOutOfRangeChannel`,
  `Geometry.Ops.WidenedOptions.BooleanAllowEmptyResultReachesTheEngine`,
  `Geometry.Ops.WidenedOptions.LayoutAndPatchBuilderRejectAnOutOfRangeChannel`,
  `geometry.uv_generation.CreatesUVLayerOnUVLessMesh`
- `actor_utils.ResolveActor.ExactPathLiveActorClasses` (EditorAssetSubsystem LoadAsset failed)
- Audio: `audio.authoring.describe_sound_wave.ReturnsDumpShape`,
  `audio.authoring.set_sound_wave_properties.RefreshesCompressionQuality` / `.RoundTrip`
  (payload failed to parse as a wave)
- Chooser: `chooser.BoolClassRoundTrip`, `chooser.SetCellInvalidValueDoesNotResizeRows`
- Data tables, all "SaveLoadedAsset failed: Asset is not registered": `data_table.add_row.DuplicateRejected` /
  `.RoundTrip`, `data_table.remove_row.PresentThenAbsent`,
  `data_table.set_row.AtomicOnLateConversionFailure` / `.CreateIfMissing` / `.UnmatchedKeysRejected` /
  `.UpdatesExisting`, `data_table.set_row_struct.ForceClearsRows` / `.RejectedWithoutForce`
- Wiki: `infra.wiki_disk_generator.PruneOnlyManifestOwnedFiles` / `.WriteFailureSkipsPruneAndManifestCommit`
- Insights: `insights.export_trace.NoGameFramesAreTypedFailure`,
  `insights.export_trace.WriteFailureIsTypedAndLeavesNoPartialFile` (it declares some expectations),
  `insights.resolve_path.AfterFileTraceReturnsPath`
- PCG, all "Component has invalid bounds": `pcg.generate.AttributesInstancesByMesh` /
  `.CountsSpawnedInstancesNotBareZero` / `.NoScheduleResolvesToken` /
  `.NonPointOutputIsDistinguishableFromNothing` / `.TicketedKickoffDoesNotBlock`
- Pose search: `pose_search.CreateDatabaseMalformedAssetPathIsRefused` /
  `.CreateSchemaMalformedAssetPathIsRefused`
- `utils.asset_dump_writer.CommitFailureRollsBack`
- `world.precondition.DispatchContract`
- World partition: `world_partition.cleanup_invalid_datalayers.ReadbackContract`,
  `world_partition.mutations.TransientWorldDataLayerReadback`,
  `world_partition.set_datalayer.ReadbackContract`

Tests that ran before the first setter in that log are not covered by this prediction.

## History
- `#1-static-flag-leak` `OPEN` reporter — Filed from the verifier's filtered run: five tests failed on engine errors that full runs had hidden, because a static suppression flag set by an earlier test was never reset. Cost: every full-suite verdict to date is unverified for engine log errors.
- `#2-per-test-reset-and-five-causes` `IN-REVIEW` developer — (1) `Tests/AutomationSuiteMaintenance.cpp` `OnTestStart` (`OnTestStartEvent`, before `RunTest`, every test) sets all three statics to false and re-runs `LoadDefaultLogSettings`, so each test starts from the configured defaults. UE 5.8's per-test mechanism is overriding `SuppressLogErrors()` / `SuppressLogWarnings()`, which `IMPLEMENT_SIMPLE_AUTOMATION_TEST` classes cannot do, so the existing setters stay and are now scoped by the reset. (2) Per test: **AnimSequence dump ×2** — fixture defect; `NewTransientAnimSequence` never initialized the data model, so reading its length hit `ValidateSequencerData` "No Movie Scene found" (`AnimSequencerDataModel.cpp:1111`). Both tests now call `GetController().InitializeModel()`, as every real sequence has; `TestAnimSequenceDumpBuilder.cpp`. **ActorGetComponentsReadsTransientAssetInPie** — real production defect; `McpActorUtils::ResolveActorFiltered` (`Utils/ActorUtils.cpp`) called `UEditorActorSubsystem::GetAllLevelActors` and `UEditorAssetLibrary::DoesAssetExist` during PIE. Both are PIE-gated by `EditorScriptingHelpers::CheckIfInEditorAndPIE`, which returns nothing and logs "The Editor is currently in a play mode." on every actor lookup in PIE. Both are now skipped while `PinWrightPieState::IsPlayInEditorActive()`: same results, no engine error. **AddMetaSoundVariableIntAlias** — expected; the test deliberately feeds raw "Int" to prove the engine rejects it, and now declares both engine errors with `AddExpectedErrorPlain` (exact text, count 1 each). **RemoveMetaSoundNodeAndDisconnect** — test defect; it asked for class `None.Metasound.Sine`, which never exists, so it always skipped its assertions. It now uses the registry key `UE.Sine.Audio`, which resolved in the same run for the sibling test. `-SingleFile` compile of all five files: clean.
- `#3-anim-fixture-needs-skeleton` `IN-REVIEW` developer — The full offscreen suite with errors unmasked (`Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log:13730-13756`) showed `InitializeModel()` on the skeleton-less `NewTransientAnimSequence` fixture logging three setup errors: "Unable to retrieve target USkeleton", "Unable to initialize FKControlRig ... provided FKControlRig is invalid" and "Unable to retrieve valid URigHierarchy". These are fixture setup errors, not expected behaviour. `WritesAnimSequenceAspectFile` and `DumpBuilder.Shape` now bind a one-bone transient skeleton (`NewTransientSkeletonWithBones({root})`, scoped root) before `InitializeModel`, the same order as the passing `BoneTracksReadback`. `TestAnimSequenceDumpBuilder.cpp` only; the shared header is untouched. `-SingleFile` compile: clean.
