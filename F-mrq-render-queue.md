---
id: F-mrq-render-queue
title: "Movie Render Queue (MRQ) job submission RPCs"
status: DONE
severity: Low
category: feature
tags: [mrq, movie-pipeline, rendering, sequencer, async, jobs]
---

# Movie Render Queue (MRQ) job submission RPCs

The plugin currently has zero references to `MoviePipeline*`, `MovieRenderQueue`,
or `MovieGraph` — there is no way to drive an MRQ render from an RPC client.
Cinematic and trailer workflows that script sequence rendering have to fall back
to console commands or hand-driven editor UI.

Proposed surface (new `mrq` namespace):

- `mrq.create_job` — params: `sequencePath` (LevelSequence asset), `levelPath`
  (map to load), `outputDir` (filesystem), `presetPath`
  (`MoviePipelinePrimaryConfig` / `MoviePipelineMasterConfig` asset). Returns
  the queued job descriptor.
- `mrq.run_jobs` — kicks the executor, returns a `jobId` from `FJobRegistry`;
  callers poll via `system.job_status` until completion. Long-running, must use
  the existing `Ctx.StartJob()` helper (see `F-handler-context-startjob`,
  `F-job-registry`).
- `mrq.list_presets` — enumerates `MoviePipelinePrimaryConfig` assets in the
  project for client discovery.

UE 5.6 implementation pattern: `UMoviePipelineQueueSubsystem::GetQueue()` to
mutate the queue, allocate a `UMoviePipelinePIEExecutor` (or `*RenderExecutor`
for `-game` mode) via `UMoviePipelineQueueSubsystem::RenderQueueWithExecutor`,
and bind `OnExecutorFinished` to call `FJobRegistry::Get().CompleteJob(...)`.
Module deps: `MovieRenderPipelineCore`, `MovieRenderPipelineEditor`,
`MovieRenderPipelineSettings`.

**Workaround:** drive renders via `editor.run_console_command` with
`MovieRenderPipeline` console commands, or invoke an editor utility blueprint.
Neither integrates with the job registry.

**Fix:** Implement `mrq.*` handlers in `Private/Handlers/MRQ/` using
`REGISTER_RPC_HANDLER`, gated by `__has_include(<MoviePipelineQueueSubsystem.h>)`
plus a `TryAddConditionalModule("MovieRenderPipelineCore")` call in
`EditorAutomationRpcGateway.Build.cs` so the plugin still compiles when MRQ is
disabled.

## History
- `#1-no-mrq-coverage` `OPEN` reporter — Plugin has no MoviePipeline handlers; cinematic/trailer workflows can't script renders. Proposing `mrq.create_job` / `mrq.run_jobs` / `mrq.list_presets` built on `UMoviePipelineQueueSubsystem` + `FJobRegistry` for long-running progress reporting.
- `#2-implement-mrq-handlers` `IN-REVIEW` developer — Added `mrq.create_job` / `mrq.run_jobs` / `mrq.list_presets` in `Private/Handlers/MRQ/MRQHandler.cpp` (new file + new directory). Gated on `__has_include` for `MoviePipelineQueueSubsystem.h` + 3 sibling headers; handlers always register so discovery is stable, but emit `MRQ_NOT_AVAILABLE` when MRQ isn't compiled in. `mrq.run_jobs` is long-running and uses the `Ctx.StartJob(Args)` pattern with completion delivered through `UMoviePipelineExecutorBase::OnExecutorFinished()`. `mrq.list_presets` queries `UMoviePipelinePrimaryConfig` via the asset registry (note: `UMoviePipelineMasterConfig` was renamed in UE 5.2 and is gone from supported versions). Added 3 `TryAddConditionalModule` lines in `Build.cs` for `MovieRenderPipelineCore`/`MovieRenderPipelineEditor`/`MovieRenderPipelineSettings`. Two `Tests/Media/TestMRQHandlers.cpp` dispatcher tests, gated on the same `MCP_HAS_MRQ` define.
- `#3-skip-editor-offline` `SKIP` tester — Unreal Editor is not running (127.0.0.1:19880 and :19882 both refuse connections); cannot exercise `mrq?` discovery or call `mrq.list_presets` / `mrq.create_job` to verify the new handlers register and respond. Protocol forbids restarting the editor, so leaving IN-REVIEW for re-verification once the editor is up.
- `#4-verify-mrq-handlers` `DONE` tester — Verified: `mrq?` discovery lists all three methods (`create_job`, `list_presets`, `run_jobs`) with the documented summaries; `mrq.list_presets` returned 4 real `UMoviePipelinePrimaryConfig` assets (count=4, e.g. `/Game/ArchvisProject/MRQ_Presets/Still_Render_Settings`); `mrq.create_job?` exposes the expected schema (`sequencePath` req, `levelPath` req, `presetPath` opt, `jobName` opt). Handlers registered, MRQ modules compiled in.
