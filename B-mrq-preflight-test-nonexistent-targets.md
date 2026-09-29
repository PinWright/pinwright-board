---
id: B-mrq-preflight-test-nonexistent-targets
title: "mrq.create_job.PreflightDisclosesTheResolvedConfig queues a job on a sequence and map that do not exist, so create_job answers ASSET_NOT_FOUND and the test skips blaming the preset"
status: IN-REVIEW
severity: Medium
category: bug
tags: [gap-analysis-2026-09-28, testing, mrq, fixture-skip, misattributed-skip]
encounters: 1
lastSeen: 2026-09-29T17:16:39Z
---

# Pre-flight test fixture points at nonexistent sequence and map

`PinWright.mrq.create_job.PreflightDisclosesTheResolvedConfig` (`Tests/Media/TestMRQHandlers.cpp`) skips with
`reason=mrq_in_memory_preset_unresolvable -- mrq.create_job refused the in-memory fixture preset [ASSET_NOT_FOUND]`
(suite log `Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log:42439`).

The skip text misattributes the refusal. The preset resolves: `mrq.create_job` loads `presetPath` with
`LoadObject`, which finds the in-memory `UMoviePipelinePrimaryConfig`, and a preset failure would be
`MRQ_PRESET_NOT_LOADABLE`, not `ASSET_NOT_FOUND`. `ASSET_NOT_FOUND` comes from the next two checks
(`Handlers/MRQ/MRQHandler.cpp:618-646`), which resolve `sequencePath` and `levelPath`; the test passed
`/Game/PinWrightTests/Seq_<guid>` and `/Game/PinWrightTests/Map_<guid>`, which never exist. The verb's
contract (refuse an unresolvable target) is correct; the fixture is incomplete. The sibling
`DisclosesPreviouslyQueuedJobs` already builds an in-memory `ULevelSequence` and a transient `UWorld` and passes.

## History
- `#1-asset-not-found-is-sequence-not-preset` `OPEN` reporter — Skip blames the in-memory preset; the `ASSET_NOT_FOUND` is from the nonexistent `sequencePath` / `levelPath` the fixture passes. Test defect, no product change.
- `#2-real-sequence-and-map-fixtures` `IN-REVIEW` developer — The test now creates an initialized in-memory `ULevelSequence` and a `UWorld::CreateWorld` map under `/Temp/PinWrightTests/Preflight_<guid>_*` (guarded by `TStrongObjectPtr` / `FScopedTransientWorldGuard`, the `DisclosesPreviouslyQueuedJobs` shape) and passes their paths; a `create_job` refusal is now a failure naming the error code and message instead of the `mrq_in_memory_preset_unresolvable` skip; the `bSuppressLogs` comment updated. `-SingleFile` compile succeeded; not run.
