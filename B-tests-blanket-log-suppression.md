---
id: B-tests-blanket-log-suppression
title: "116 blanket bSuppressLogErrors / bSuppressLogWarnings setters in 30 test files switched off error checking for whole tests"
status: IN-REVIEW
severity: High
category: bug
tags: [tests, log-suppression, false-green, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T16:40:00Z
---

# Blanket log suppression in tests

After `B-suppress-log-errors-static-leaks` scoped the static suppression flags to one test, each
test that set them still ignored every engine Error (or Warning) it caused, including errors from
verb or fixture defects a user would also hit. The audit covered every
`bSuppressLogErrors = true` / `bSuppressLogWarnings = true` under `Source/**/Tests`: 116 setter lines in
30 files. For each site it read the Error/Warning lines the test logged in the full offscreen run
`Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log`, where the lines are
still visible.

**Fix:** see History.

## History
- `#1-audit` `OPEN` reporter — Audit requested after the drive.observe case (`B-simulate-input-pie-runs-host-gamemode` #4) showed a blanket setter hiding a real host error.
- `#2-converted-and-ratcheted` `IN-REVIEW` developer — Decisions per file:
  - **Removed, test logged no Error at all:** TestActorSpawnMaterialAssignment (2), TestSpawnMaterialSurvivesConstructionScript (1), TestAssetDeleteForceGate (1), TestAssetHandlers (2), TestAssetMoveDestinationFolder (2), TestAssetRenameDuplicateVerification (4), TestBlueprintHandlers (2), TestAnnotatedCaptureHandlers (2), TestActorHandlers (5 of 6), TestGeometrySkeletalMeshRoundTrip (4), TestGeometryStaticMeshRoundTrip (1), TestMeshMeasureHandler.NotFound (1), TestPlacementHandlers.TypedRefusals (1), TestParamTypeGate.AcceptsLosslessCoercions (1), TestDispatcher.RejectsNullPayload (2). All drive tests too, which logged nothing: TestDriveActionHandlers (16), TestDriveEditorIntegration (8), TestDriveHandlersSync (8), TestDriveWebHandlers (20), TestDriveWindowSelectorParams (2). Any Warning lines these tests log were never affected by an errors-only setter.
  - **Replaced by exact expectations** (`AddExpectedErrorPlain` / `AddExpectedMessagePlain`, bare message, exact count; these are the refusals the tests deliberately provoke): TestDispatcher `ExceptionHandling.StdException` / `.Unknown` (the two handler-exception Errors) and `UnknownParamsDiscoveryGuidance`. Dispatcher-gate Warnings: TestNestedParamKeyGate (4 tests, including the engine "Failed to find object" lines of the sibling cases that do load), TestParamTypeGate (7), TestPathParamSeparatorGate (4), TestMaterialHandlers (2), TestNoParamHandlersRejectUnknownArgs, spatial measure/place_relative/raycast/raycast_screen UnknownArgRejected, TestWidgetSetSlotDispatch, TestMeshMeasureHandler.RejectsUnknownParam. `Sequencer.ControlRigBake.AnimSequenceRoundTrip` declares the engine's one `Can not open Sequencer for the LevelSequence None` Error, only when no Level Sequence is current (the engine's own condition). `RefusesNonSkeletalBinding` logged nothing and was removed.
  - **Root cause fixed instead:** `actor.spawn_from_blueprint.WithPath` hid `LoadAsset failed ... could not be found` from `ResolveClassByName`, a defect users also hit. Ticket `B-resolve-class-by-name-logs-loadasset-error`.
  - **Left blanket:** none.
  - **Guard:** `PinWright.infra.contract.LogSuppression.NoUnlistedBlanketSetters` (`Tests/Infra/TestBlanketLogSuppressionRatchet.cpp`) scans every test source and fails on any assignment of the three statics beyond a per-file allowance (currently empty), or when a listed allowance is stale. `AutomationSuiteMaintenance.cpp` (the reset) is exempt.
  - **Verify:** the counts come from one full run. If one differs on the next run, the failure names the pattern with expected vs. found; that is a real change in what the test provokes, not flake.
