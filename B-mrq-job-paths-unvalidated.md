---
id: B-mrq-job-paths-unvalidated
title: "mrq.create_job queues unresolved sequence and map paths as success, and the PIE executor can later finish the rejected job with success:true"
status: IN-REVIEW
severity: High
category: bug
tags: [mrq, queue, sequence, map, validation, false-success]
encounters: 1
lastSeen: 2026-09-03T23:08:58+03:00
---

# MRQ accepts invalid render targets and can later report a successful no-op

## What happens

`mrq.create_job` requires two non-empty strings but does not resolve either target.
After validating only the optional preset, it allocates a shared-queue job and
assigns raw `FSoftObjectPath` values
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\MRQ\MRQHandler.cpp:355-358,387-433`),
then echoes both strings in a success response at `:447-488`.

The default PIE executor later calls `InJob->Sequence.TryLoad()` and, when that
fails, calls `OnExecutorFinishedImpl()` without recording an error
(`C:\UE_5.8\Engine\Plugins\MovieScene\MovieRenderPipeline\Source\MovieRenderPipelineEditor\Private\MoviePipelinePIEExecutor.cpp:97-112`).
The invalid/unsaved-map branch does the same at `:115-129`. The base implementation
derives success only from whether a fatal error was recorded
(`C:\UE_5.8\Engine\Plugins\MovieScene\MovieRenderPipeline\Source\MovieRenderPipelineCore\Public\MoviePipelineExecutor.h:245-268`),
so this rejection can reach `mrq.run_jobs` as `success:true` with no render artifact.

## Why it matters

One typo creates a persistent trap in the editor-global render queue. The create
call says the job was accepted, and a later run can also say it succeeded even
though the executor rejected the job before rendering. That is a silent wrong
result on the normal authoring path, so severity is High.

## What should happen

Resolve and type-check `sequencePath` as `ULevelSequence` and `levelPath` as a
saved map before `AllocateNewJob`, matching the existing preset preflight shape in
this handler. Return a typed validation error and leave the shared queue unchanged.
The run completion path should also refuse success when no requested job produced
work because target validation stopped the executor.

## Workaround

Resolve both assets independently before calling `mrq.create_job`, then inspect
`mrq.list_jobs` and require measured output artifacts after `mrq.run_jobs`.

## Fix

Verdict: TRUE (source-only verification; no build, editor, MCP, or runtime reproduction was
performed). The root cause was that `mrq.create_job` treated the required target strings as opaque
values, assigned them directly to `FSoftObjectPath`, and only then allocated a shared-queue job;
the completion path also trusted the executor's success bit without measuring output.

The handler now validates `sequencePath`, `levelPath`, and an optional `presetPath` before queue
work using `FPackageName::IsValidObjectPath`, `ObjectPathToPackageName`,
`IsValidLongPackageName`, and `GetPackageMountPoint`, then resolves the sequence and map through
the registry-backed `Utils/AssetUtils` resolver, requiring `ULevelSequence` and a loaded `UWorld`
map asset, including transient in-memory maps used by tests.
Missing and wrong-type targets return `ASSET_NOT_FOUND` and `ASSET_WRONG_TYPE` before allocation. It
also validates the resolved preset/default `OutputDirectory` before allocation: `{project_dir}` is expanded, traversal and file-valued paths
are refused (including file-valued ancestors), and an `IFileManager::CreateFileWriter` probe verifies
a writable existing ancestor so new nested directories remain valid. Every refusal uses
`ErrorCodes::ERR_INVALID_PATH`; queue size is checked by the tests before and after the request.
The PIE completion path now returns `RENDER_NO_OUTPUT` when it reports success without measured
output.

Files changed:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/MRQ/MRQHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Media/TestMRQHandlers.cpp`
- `Plugins/PinWright/Docs/wiki-src/mrq.md`
- `Plugins/PinWright/Source/PinWright/Private/Handlers/ErrorCodes.h`
- `Plugins/PinWright/Source/PinWright/Private/Utils/AssetUtils.h`
- `Plugins/PinWright/Source/PinWright/Private/Utils/AssetUtils.cpp`

Behavioral test IDs added:

- `PinWright.mrq.create_job.NonexistentSequenceDoesNotMutateQueue`
- `PinWright.mrq.create_job.WrongTypeSequenceDoesNotMutateQueue`
- `PinWright.mrq.create_job.InvalidLevelPathDoesNotMutateQueue`
- `PinWright.mrq.create_job.EmptyOutputDirectoryDoesNotMutateQueue`
- `PinWright.mrq.create_job.FileValuedOutputAncestorDoesNotMutateQueue`
- `PinWright.mrq.create_job.GuidNamedNonexistentAssetsAreRefused`

Deliberate nonchanges: no build, test, editor, MCP, or reproduction run was performed; non-PIE
executors still disclose that their output is unmeasured rather than claiming no output.

## Related

- Data-loss pattern `wrong-target-identity-or-fallback`.
- False-success patterns `wrong-target-scope-or-identity` and `request-echo-not-result-readback`.
- `B-mrq-shared-queue-no-management-verbs` — shared-queue ownership sibling.
- `B-mrq-run-jobs-succeeds-on-unrenderable-frames` — different artifact-validity failure.

## History
- `#1-filed-pattern-scan` `OPEN` reporter — Source-only scan followed both inputs from `MRQHandler.cpp:355-358` through unvalidated assignment at `:425-433` and the success response at `:447-488`, then confirmed the PIE executor's invalid-sequence and invalid-map exits call the success-derived finish path without a fatal error. No build, test, editor, MCP call, plugin edit, commit, or repro was performed.
- `#2-validated-paths` `IN-REVIEW` implementer — Added mounted long-package/object-path validation for MRQ targets, writable-ancestor probing for resolved output directories, typed `INVALID_PATH` refusals before job allocation, focused queue-unchanged harness tests, and the namespace contract note. No build, test, editor, MCP call, or runtime reproduction was performed.
- `#3-output-ancestor-audit` `IN-REVIEW` developer — Source-only audit found that a file-valued intermediate output-path component could be skipped while searching for a writable ancestor; added an early `INVALID_PATH` refusal and the behavioral queue-unchanged test `PinWright.mrq.create_job.FileValuedOutputAncestorDoesNotMutateQueue`. Also restored the `levelPath` parameter declaration to type `path`. No build, test, editor, MCP call, or runtime reproduction was performed.
- `#4-asset-resolution-and-output-measurement` `IN-REVIEW` implementer — Corrected verifier findings: create now resolves registry-backed targets and rejects missing/wrong-type sequence or map before queue allocation; PIE completion reports `RENDER_NO_OUTPUT` when success has no measured output; traversal rejects only full `..` components; required tests cover missing and wrong-type assets and the prior GUID-shaped success case; affected fixtures use `TStrongObjectPtr`. Wiki contract updated. No build, test, editor, MCP call, or runtime reproduction was performed.
- `#5-canonical-object-path-audit` `IN-REVIEW` developer — Source-only audit found valid package-only inputs were resolved to canonical object paths but the queued job still stored the raw package-only strings; `create_job` now stores the resolver's canonical sequence/map object paths. No build, test, editor, MCP call, or runtime reproduction was performed.
- `#6-suite-fixture-correction` `IN-REVIEW` implementer — Suite evidence showed `DisclosesPreviouslyQueuedJobs` still supplied GUID-shaped nonexistent create targets, so the new resolver correctly returned `ASSET_NOT_FOUND` and the old success assertion was invalid. TEST-CORRECTED the fixture to use shipped `ULevelSequence`/map assets while retaining synthetic pre-existing list-disclosure data. No build, test, editor, MCP call, or runtime reproduction was performed.
- `#7-standalone-transient-mrq-fixture` `IN-REVIEW` implementer — Corrected the host-content regression by replacing shipped assets with test-owned transient `ULevelSequence` and `UWorld` fixtures under `/Temp/PinWrightTests/<GUID>`, guarded by `TStrongObjectPtr` and `FScopedTransientWorldGuard`. The handler accepts loaded transient map assets; no raw root operations were added. No build, test, editor, MCP call, or runtime reproduction was performed.
- `#8-mrq-sequence-initialization-crash` `IN-REVIEW` implementer — Crash evidence showed `ULevelSequence::GetMovieScene()` was null during `UMoviePipelineExecutorJob::SetSequence`; initialized the transient fixture and added a pre-allocation `SEQUENCE_INVALID` refusal plus queue-unchanged coverage for uninitialized sequences. No build, test, editor, MCP call, or runtime reproduction was performed.
