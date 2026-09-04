---
id: B-mrq-abstract-executor
title: "mrq.run_jobs accepts the abstract MoviePipeline executor base class and reaches its fatal pure-virtual Execute implementation"
status: IN-REVIEW
severity: Critical
category: bug
tags: [mrq, executor, validation, abstract-class, editor-crash]
encounters: 1
lastSeen: 2026-09-03T23:08:58+03:00
---

# An abstract MRQ executor class can terminate the editor

## What happens

`mrq.run_jobs` accepts any class that `LoadClass<UMoviePipelineExecutorBase>` can
load and checks only for null
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\MRQ\MRQHandler.cpp:604-615`).
The deferred job then constructs that class at `MRQHandler.cpp:739-745` and starts
it at `:833`. A caller can therefore pass
`/Script/MovieRenderPipelineCore.MoviePipelineExecutorBase`: the engine declares
that class `Abstract` and its `Execute_Implementation` is `PURE_VIRTUAL`
(`C:\UE_5.8\Engine\Plugins\MovieScene\MovieRenderPipeline\Source\MovieRenderPipelineCore\Public\MoviePipelineExecutor.h:45,278`).
In an editor build, `NewObject` reaches the engine's abstract-class ensure and
construction continues; queue execution then invokes the fatal pure-virtual body.

## Why it matters

A syntactically valid RPC parameter can terminate the editor process. The caller
loses the active session and any unsaved editor state, with no RPC error response.
This is Critical under the board rubric because the concrete base-class path
reaches a fatal implementation rather than merely producing an unusable object.

## What should happen

Before starting the job, reject classes with `CLASS_Abstract` using the validated
class-selection shape already requested by
`B-pcg-add-node-no-abstract-class-guard`. Return a typed
`INVALID_EXECUTOR_CLASS` error without constructing the class or changing the
queue. Also reject deprecated/newer-version classes if the shared class guard does.

## Workaround

Omit `executorClass` to use `UMoviePipelinePIEExecutor`, or pass a known concrete
`UMoviePipelineExecutorBase` subclass.

## Related

- Crash pattern `validate-before-checked-builder-or-factory`.
- `B-pcg-add-node-no-abstract-class-guard` — sibling class-validation fix shape.
- `B-mrq-shared-queue-no-management-verbs` — shared-queue behavior, not this crash.

## Fix

The root cause was that `mrq.run_jobs` used the caller's loadable class immediately after a null
check, so the abstract `UMoviePipelineExecutorBase` passed into the deferred `NewObject` path.
`MRQHandler.cpp` now resolves the name independently, preserves `CLASS_NOT_FOUND` for an
unloadable path, and rejects an abstract class or a class outside the
`UMoviePipelineExecutorBase` hierarchy with `INVALID_EXECUTOR_CLASS` before `StartJob`.

Changed `Source/PinWright/Private/Handlers/MRQ/MRQHandler.cpp`, registered the new code in
`Source/PinWright/Private/Handlers/ErrorCodes.h`, added
`PinWright.mrq.run_jobs.RejectsInvalidExecutorClasses` in
`Source/PinWright/Private/Tests/Media/TestMRQHandlers.cpp`, and documented the refusal contract in
`Docs/wiki-src/mrq.md`. The test calls the handler with the abstract engine base and a unique bogus
class name, asserting their distinct typed errors and unchanged queue length. No other class-name
parameters exist in `MRQHandler.cpp`; queue selection, render lifecycle, and deprecated/newer-class
policy were deliberately unchanged because the referenced sibling guard covers abstractness only.

## History
- `#1-filed-pattern-scan` `OPEN` reporter — Source-only scan confirmed the caller-controlled class load at `MRQHandler.cpp:604-615`, construction at `:739-745`, start at `:833`, and the engine base class's abstract declaration plus fatal `PURE_VIRTUAL` Execute body. The board had no ticket for this MRQ mechanism. No build, test, editor, MCP call, plugin edit, commit, or repro was performed.
- `#2-validate-executor-class` `IN-REVIEW` developer — Resolved `executorClass` before ticket creation and rejected abstract or wrong-hierarchy classes with registered `INVALID_EXECUTOR_CLASS`, preserving `CLASS_NOT_FOUND` for unloadable paths; updated `MRQHandler.cpp` and the MRQ wiki, and added `PinWright.mrq.run_jobs.RejectsInvalidExecutorClasses` covering the abstract base and a bogus class without starting PIE.
