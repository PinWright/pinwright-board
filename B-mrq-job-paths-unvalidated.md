---
id: B-mrq-job-paths-unvalidated
title: "mrq.create_job queues unresolved sequence and map paths as success, and the PIE executor can later finish the rejected job with success:true"
status: OPEN
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

## Related

- Data-loss pattern `wrong-target-identity-or-fallback`.
- False-success patterns `wrong-target-scope-or-identity` and `request-echo-not-result-readback`.
- `B-mrq-shared-queue-no-management-verbs` — shared-queue ownership sibling.
- `B-mrq-run-jobs-succeeds-on-unrenderable-frames` — different artifact-validity failure.

## History
- `#1-filed-pattern-scan` `OPEN` reporter — Source-only scan followed both inputs from `MRQHandler.cpp:355-358` through unvalidated assignment at `:425-433` and the success response at `:447-488`, then confirmed the PIE executor's invalid-sequence and invalid-map exits call the success-derived finish path without a fatal error. No build, test, editor, MCP call, plugin edit, commit, or repro was performed.
