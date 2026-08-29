---
id: B-mrq-render-result-omits-bitrate-and-size
title: "`mrq.run_jobs` resolves to `{\"success\":true}` and publishes nothing about the file it produced — a 1080p60 cinematic encoded at 1.16 Mbps passes every reported check (resolution, frame count, duration, container) and ships visibly banded"
status: IN-REVIEW
severity: High
category: bug
tags: [mrq, run_jobs, create_job, movie-pipeline, mp4, encoder, bitrate, crf, rate-control, silent-wrong-output, no-readback, response-honesty, deliverable, cinematic, verification-gap, measured-vs-requested]
encounters: 1
lastSeen: 2026-08-28T00:00:00+05:00
---

# The render succeeded, every number it reported was correct, and the file was broken

A 20-second 1080p60 cinematic rendered through `mrq.create_job` + `mrq.run_jobs` came out visibly
bad — heavy banding, mushy gradients. Probed from the finished file:

| | delivered | re-render |
|---|---|---|
| video bitrate | **1,160,415 bps** | 21.24 Mbps |
| overall bitrate | 1,359,637 bps | — |
| size | **3.4 MB** | 51.2 MB |
| resolution | 1920x1080 | 1920x1080 |
| frames | 1201 | 1201 |
| frame rate | 60/1 | 60/1 |
| duration | 20.033 s | 20.033 s |
| container | valid `ftypmp42` | valid `ftypmp42` |

Same sequence, same map, same everything but the rate-control mode: **18x the bitrate for identical
content**. Resolution, frame count, duration and container were all explicitly verified against the
finished file *before* the banding was noticed by eye. Every one of them passed. The one number that
separated the good file from the bad one — bits per second — is not reported by any PinWright verb,
so there was nothing to check it against.

Immediate cause is the engine default, and the scene is its pathological case:
`UMoviePipelineMP4EncoderOutput` ships `EncodingRateControl = Quality` with `ConstantRateFactor = 20`
(`C:/UE_5.8/Engine/Plugins/MovieScene/MovieRenderPipeline/Source/MovieRenderPipelineMP4Encoder/Public/MoviePipelineMP4EncoderOutput.h:51`
and `:79`). The shot is heavy volumetric fog over dark blue gradients — large smooth areas, almost no
high-frequency detail. Quality-targeted encoding allocates bits where detail is; there is little
detail, so it allocates almost none, and banding in near-flat gradients is precisely what a viewer
notices. Re-rendered with `VARIABLE_BIT_RATE`, `AverageBitrateInMbps = 50`, max 100, it looks right.

## What the plugin reports today (source review)

`Plugins/PinWright/Source/PinWright/Private/Handlers/MRQ/MRQHandler.cpp`:

- **`mrq.create_job` response** — `:103-110`: `jobIndex`, `sequencePath`, `levelPath`, `presetPath`,
  `jobName`, `queueSize`. Nothing about resolution, frame rate, output directory, output format or
  encoder settings.
- **`mrq.run_jobs` started payload** — `:158-160`: `queueSize` and `executorClass`, merged into the
  ticket response by `Handlers/HandlerContext.cpp:626-630`.
- **`mrq.run_jobs` terminal result** — `:182-184`, the whole of it:

  ```cpp
  TSharedPtr<FJsonObject> Result = MakeShared<FJsonObject>();
  Result->SetBoolField(TEXT("success"), bSuccess);
  OnComplete(bSuccess, Result, bSuccess ? FString() : TEXT("MRQ executor reported failure"));
  ```

  One boolean. No output path, no size, no bitrate, no duration, no frame count, no encoder echo.

Answers to the three questions this ticket was opened to settle:

1. **Does anything carry output size, bitrate or encoder settings?** No. Not one field, in any of the
   three `mrq` methods, at any stage. The result does not even carry the **path of the file it
   wrote** — a caller cannot probe the output without reconstructing the MRQ output directory
   out-of-band from the preset asset.
2. **Are the caller's encoder settings echoed back?** No, and there is nowhere for them to be echoed
   *from*: `mrq.create_job` accepts only `presetPath` (`:40`), so encoder settings never pass through
   PinWright at all. They live on the preset asset and are applied by `Job->SetConfiguration(Preset)`
   at `:95`. **Worse — `presetPath` is echoed at `:107` verbatim even when the preset did not load.**
   `:87-92` treats a failed `LoadObject` as a soft failure, logs `UE_LOG(..., Warning, ...)` to the
   editor log, and queues the job **with no configuration at all**. The response is byte-identical to
   the success case. That is this same defect a second time on the same code path: the render then
   runs on bare CDO defaults — including `Quality`/CRF 20 — and the caller's evidence says their
   preset was applied.
3. **Does PinWright set any encoder defaults of its own?** No. `grep -rn` over the entire plugin
   source for `MP4Encoder`, `EncodingRateControl`, `ConstantRateFactor`, `AverageBitrate`,
   `MoviePipelineOutputSetting` and `FindOrAddSettingByClass` returns **zero hits**. PinWright
   inherits UE's untouched.

Nothing in the ask shrinks. The response shape is emptier than the ticket assumed.

## The real defect: the reported set is complete-looking and omits the number that decides

This is not "the default is badly chosen". It is that **the call succeeded, every published signal was
correct, and the output was wrong** — because the deciding measurement was never published. A caller
who does the diligent thing (verify resolution, verify frame count against the finished file, verify
duration, verify the container magic) has exhausted the evidence PinWright offers and still ships a
broken deliverable. The verification surface is complete enough to feel sufficient, which is what
makes it dangerous; an obviously thin response would have sent the caller to a probe.

### Ask 1 (primary): report the achieved encode, and warn when it is implausible

On ticket resolution `mrq.run_jobs` should carry, per rendered job, the facts about the artifact:

- `outputPath` / `outputFiles[]` — resolved from the job's `UMoviePipelineOutputSetting`
  (`OutputDirectory` + `FileNameFormat`); the single most-missing field here.
- `fileSizeBytes` — a `stat`.
- `durationSeconds`, `frameCount`, `resolution`, `frameRate` — already knowable from the config and
  the sequence; cheap, and they make the response self-checking.
- `videoBitrateBps`, or `overallBitrateBps` = `fileSizeBytes * 8 / duration`, which needs no demuxing
  at all and would have caught this case on its own.
- `encoder: { class, rateControl, constantRateFactor, averageBitrateMbps, maxBitrateMbps }` read back
  off the resolved output setting — the *effective* values, not what the caller believes.

And a **plausibility warning**, which is the part that turns data into a signal. Express the floor per
pixel so it holds at any resolution and frame rate:

    bitsPerPixel = bitrateBps / (width * height * fps)

- delivered file: 1,160,415 / (1920*1080*60) = **0.0093 bpp**
- re-render:      21,240,000 / (1920*1080*60) = **0.171 bpp**
- a floor at **0.04 bpp** = 4.98 Mbps at 1080p60, i.e. the "roughly 5 Mbps" smell test, and it
  generalises: 1.24 Mbps at 720p30, 19.9 Mbps at 4K30, no table needed.

The bad file sits **4.3x below** that floor. Emit `warnings: ["encoded at 0.009 bits/pixel (1.16 Mbps
at 1920x1080@60) — 4.3x below the 0.04 bits/pixel plausibility floor; rate control is Quality/CRF 20,
which has no lower bound on smooth content"]`. One `stat`, one divide, no decoding, no new dependency.
It does not block the render and it does not have to be right about *why* — it only has to make the
caller look.

### Ask 2 (secondary): the encoder default

This was raised proposing a **bitrate floor under the quality target** — keep quality-targeting's
adaptiveness so a simple scene still yields a small file, remove the pathological low end. That is the
right *shape*, and on this encoder it is **not implementable**, which the source settles:
`MovieRenderPipelineMP4Encoder/Private/Windows/MoviePipelineMP4Encoder.cpp:749-763` — the `Quality`
branch sets only `CODECAPI_AVEncVideoEncodeQP` and `CODECAPI_AVEncCommonQuality = 0`, and **never
sets `AVEncCommonMeanBitRate` or `AVEncCommonMaxBitRate`**, unlike every other branch (`:709-748`).
Media Foundation's quality mode has no floor input to attach. There is no PinWright-side way to bound
it from below without changing the rate-control mode — and the default itself is Epic's CDO, in engine
code this plugin does not own, so "change our default" is not on the table either.

The honest remedy, argued in place of the one proposed:

- **Pre-flight disclosure at `mrq.create_job`.** Read the resolved primary config and echo the video
  output class and its effective rate-control settings in the response, with a warning when the mode
  is `Quality` or `ConstantQP` — *unbounded below on low-detail content; consider VariableBitRate*.
  This is where being wrong is cheapest: the caller finds out before spending the render, not after.
  It is also where the `:87-92` silent preset-drop must stop being silent — a `presetApplied: false`
  field, or a hard `PRESET_NOT_LOADABLE` error, so `presetPath` in the response stops being a claim
  the handler never checked.
- **Optional `minBitsPerPixel` (or `minBitrateMbps`) param on `mrq.create_job`.** When set and the
  resolved config is quality-targeted, rewrite the output setting to `VariableBitRate` with
  `AverageBitrateInMbps` derived from the floor, and say so in the response. Opt-in, so a caller who
  wants CRF keeps CRF.
- If the fix budget covers only one thing, **it is ask 1.** A bad default that reports itself is a
  nuisance; a bad default that reports nothing is this ticket.

## This is the plugin's recurring failure shape, not an `mrq` quirk

**The call succeeds, every number that is reported is correct, and the output is wrong because the
number that mattered was never reported.** The session that hit this hit the same shape twice more:

- `B-ground-probe-hits-hull-not-render` (High) — a hull hit and a render-surface hit are
  indistinguishable in the response; a prop seated on a phantom surface floats 202 cm and every
  reported number looks right.
- `B-niagara-validate-green-while-component-inactive` (High) — `valid:true` for a system no placed
  component is running; no verb reports activation.
- this ticket — `success:true` for an encode nothing measured.

The shape is already on the board twice: `B-thumbnail-cold-first-frame-no-stats` (High, DONE — a cold
frame and a finished frame return the same success payload because no image statistics are published)
and `E-property-set-no-measured-render-state` (Medium — the write is notified; whether the renderer
took it is unreported). The common defect is **a response whose field set is complete enough to look
like verification while omitting the one measurement that separates success from failure**. The
general rule — a verb that produces an artifact must publish a measurement *of the artifact*, not only
of the request — belongs in the plugin's response-honesty guidance rather than in any single handler;
three High tickets in one session is the evidence for stating it once, centrally.

## Workaround

Probe the file yourself and do not trust the ticket. The output path is in no response, so reconstruct
it from the preset's `UMoviePipelineOutputSetting` (`OutputDirectory` + `FileNameFormat`), then check
`size*8/duration` against ~0.04 bits/pixel. To avoid the trap entirely, set the preset's
`MoviePipelineMP4EncoderOutput` to `VARIABLE_BIT_RATE` with an explicit `AverageBitrateInMbps` before
`mrq.create_job` — never leave a fog, night, underwater or smoke shot on the `Quality`/CRF 20 default.
And confirm the preset actually loaded (editor log `Warning: mrq.create_job: preset '...' not
loadable`), because the response says it did either way.

## Adjacent, not folded in

`mrq.run_jobs` returned a bare `{"success":true}` with **no job ticket**, though the `mrq` wiki
documents a ticket to poll via `system.job_status`. Cause is `HandlerContext.cpp:612-632`: on a
streaming request (`progressToken` + `Accept: text/event-stream` + `wait=true`) `StartJob` suppresses
the immediate ticket response entirely and resolves the HTTP request with the *terminal* result — so
the caller receives the completion payload and never sees a `ticket_id`, while the doc tells them to
poll for one. The doc-vs-result mismatch on that same aggregate payload is already filed as
`E-mrq-run-jobs-doc-promises-per-job-exit-status` (Low, OPEN); the missing-ticket half is not covered
there and belongs on that ticket or a new `E-`, not on this one.

## Severity rationale

impact=**High** by the board's table — silent wrong output on a normal path, where the caller trusts a
result that is a lie by omission and builds on it (here: ships it). No crash and no data loss, so not
Critical. reach=**rare** in the literal per-session sense (offline cinematic render), which the reach
modifier would bump to Medium; **declining the bump**, for two reasons a reviewer can check: (1) the
artifact is a *deliverable that leaves the tool* — the cost of a miss is a bad file in someone's
hands, not a retry, and every check a careful caller runs passes; (2) `mrq` is the plugin's only
offline render path, so it is rare per session but near-universal within cinematic work, which is
standing guidance to route through MRQ rather than viewport capture. Precedent supports High:
`B-thumbnail-cold-first-frame-no-stats` is the same defect (no statistics published, so a broken
artifact is indistinguishable from a good one) and was rated High. A reviewer who weights reach
strictly can downgrade to Medium knowing exactly which argument they are rejecting.

## History
- `#1-initial-repro` `OPEN` reporter — 20 s 1080p60 cinematic via `mrq.create_job` + `mrq.run_jobs`
  came out visibly banded over volumetric fog. Probed: 1,160,415 bps video / 1,359,637 bps overall /
  3.4 MB, against correct 1920x1080, 1201 frames, 60/1, 20.033 s, valid `ftypmp42` — frame count and
  duration explicitly checked against the finished file, all four passed. Re-render with
  `VARIABLE_BIT_RATE` / avg 50 / max 100 Mbps: 21.24 Mbps, 51.2 MB, identical frames and duration,
  looks right — 18x the bitrate for the same content. Source review confirms the plugin publishes
  nothing about the artifact: `MRQHandler.cpp:182-184` sets one `success` bool and no output path;
  `:103-110` echoes only queue bookkeeping; `:40` accepts no encoder params; `:87-92` drops an
  unloadable preset to a log warning while `:107` still echoes `presetPath`; zero grep hits for any
  MP4/encoder/output-setting symbol anywhere in plugin source, so UE's `Quality`/CRF 20 CDO default
  (`MoviePipelineMP4EncoderOutput.h:51`, `:79`) is inherited untouched. The proposed
  bitrate-floor-under-quality remedy is not expressible on this encoder —
  `MoviePipelineMP4Encoder.cpp:749-763` sets no mean or max bitrate in the `Quality` branch — so
  pre-flight disclosure plus an opt-in floor param is argued in its place; the reporting ask (achieved
  bitrate and size, plus a 0.04 bits/pixel plausibility warning) is unchanged and is the primary ask.
  Filed as its own ticket, not a dedup of `E-mrq-run-jobs-doc-promises-per-job-exit-status` (Low, doc
  overclaim about per-job exit status), which is cross-linked. Third instance this session of the same
  shape after `B-ground-probe-hits-hull-not-render` and
  `B-niagara-validate-green-while-component-inactive`.
- `#2-ask-1-artifact-report` `IN-REVIEW` developer — ask 1 implemented; ask 2 (pre-flight disclosure at
  `mrq.create_job`, the `:87-92` silent preset-drop, the opt-in `minBitsPerPixel` param) deliberately
  NOT touched and still owed. Every claim in the ticket's source review was re-verified against the
  tree and holds: `mrq.run_jobs`' terminal result really was one `success` bool, `create_job` echoes
  `presetPath` on the unloadable-preset path, and the plugin sets no encoder defaults. New
  `Handlers/MRQ/MRQArtifactReport.{h,cpp}` (no MovieRenderPipeline includes, so it builds and is
  tested with the engine plugin disabled) stats every file the pipeline reported writing and publishes
  `outputFiles[]` (`path`, `renderPass`, `exists`, `fileSizeBytes`), `outputFileCount`,
  `measuredFileCount`, `totalFileSizeBytes`, `frameCount`, `frameRate`, `durationSeconds`,
  `resolution`, `overallBitrateBps`, `bitsPerPixel`, `encoderRequested` and `warnings`;
  `MRQHandler.cpp` binds `UMoviePipelinePIEExecutor::OnIndividualJobWorkFinished` to collect one entry
  per job into a new `jobs` array on the terminal result. **Measured vs requested, settled against the
  source:** size and bitrate are read off disk and named plainly; the encoder read-back is named
  `encoderRequested` and never `encoder`, because `MoviePipelineMP4Encoder.cpp`'s Quality branch sets
  only an encode QP and never `AVEncCommonMeanBitRate`/`MaxBitRate` — the requested settings place no
  lower bound on what is achieved. **The file IS on disk at readback time**, which the ticket left
  open: `FMoviePipelineOutputData` is filled by `UMoviePipeline::ProcessOutstandingFutures` on the
  Finalize→Export transition under the engine's own comment "the futures won't be available until
  actually written to disk", and the PIE executor broadcasts a tick later — a reported path that is
  nevertheless absent gets `exists:false` and NO `fileSizeBytes`. Nothing unmeasurable is zeroed:
  missing duration omits `overallBitrateBps` with a warning, missing files omit their size, an
  empty output list says so, and a non-PIE `executorClass` omits `jobs` entirely in favour of
  `artifactWarning` rather than returning an empty array that would read as "wrote nothing". The
  0.04 bits/pixel floor and a `Quality`/`ConstantQP` rate-control warning are both emitted.
  Regression test `Tests/Media/TestMRQArtifactReport.cpp`, four cases under `PinWright.mrq.run_jobs.*`
  (`ArtifactSizeAndBitrateAreMeasured`, `ArtifactUnmeasurableIsOmittedNotZeroed`,
  `ArtifactPlausibilityFloorIsPerPixel`, `EncoderReadbackReportsShippedDefault`) against real files
  written to `Intermediate/` and against the engine's own `UMoviePipelineMP4EncoderOutput` CDO — which
  is what pins the shipped `Quality`/CRF-20 default rather than a value the test supplied. **Stated
  limitation:** a real MRQ render needs PIE, a saved map and minutes of wall time, so the terminal
  result assembly in `MRQHandler.cpp` is NOT driven end-to-end by the suite; the report builder that
  produces every new field is. `Docs/wiki-src/mrq.md` documents the result shape. Not compiled and not
  run — the wave owner builds.
- `#3-ask-2-preflight-and-preset-refusal` `IN-REVIEW` developer — ask 2 landed except the opt-in
  floor param, which is argued out of scope below. Status stays `IN-REVIEW`; a tester still owns both
  asks. **The silent preset-drop is fixed by REFUSING, not by a flag.** `MRQHandler.cpp` now resolves
  `presetPath` BEFORE `AllocateNewJob`, so an unloadable preset leaves the queue byte-identical and
  answers `MRQ_PRESET_NOT_LOADABLE` (registered in `Handlers/ErrorCodes.h`; the call site keeps a raw
  literal because that file is non-adopting and one `ErrorCodes::` reference would turn its other
  seven hand-spelled codes into hard failures). Chosen over `presetApplied: false` and over a warning
  for three checkable reasons: (1) the same handler already hard-errors `CLASS_NOT_FOUND` for an
  unloadable `executorClass` at `run_jobs`, so a warning here was internally inconsistent; (2) an
  unconfigured job is a supported state only when the caller asked for it by omitting `presetPath` —
  `Docs/rpc-design.md` §1, "an unresolvable target is an error, and the empty result is reserved for a
  question that was actually asked"; (3) the cost is one-directional — refusing costs a corrected path
  at queue time, proceeding costs minutes of render plus a deliverable that leaves the tool, and the
  misconfigured job would have sat in the editor-global queue as a trap for anyone's later
  `run_jobs`. **`presetApplied` was deliberately NOT added**: `UMoviePipelineExecutorJob` creates its
  `Configuration` as a default subobject (`MoviePipelineQueue.h:372`), so `GetConfiguration() != null`
  measures nothing and the boolean could only have restated the request. The pre-flight block is the
  honest replacement — it publishes the preset's own settings, so it cannot claim a preset took effect
  that did not. **Pre-flight disclosure** is `PinWrightMRQ::BuildPreflightReport` added beside the
  existing artifact report in `Handlers/MRQ/MRQArtifactReport.{h,cpp}` (same file, same no-
  MovieRenderPipeline-includes invariant), reusing `ReadRequestedEncoderSettings` and
  `RateControlIsUnboundedBelow` rather than growing a second copy; extraction is
  `ReadPreflightContext` in `MRQHandler.cpp`, read off the job's RESOLVED config after
  `SetConfiguration` copied the preset in. `create_job` now carries `preflight` — `resolution`,
  `outputDirectory` and `fileNameFormat` (published UNRESOLVED, because MRQ expands `{project_dir}` /
  `{sequence_name}` / `{frame_number}` at render time from state that does not exist at queue time, and
  a resolved-looking filename would be a claim about a file nothing has computed), `frameRateOverride`
  only when the config overrides the sequence rate, `outputs[]` (class path of every ENABLED file
  writer; empty means this job writes nothing) and `encoderRequested` — plus a top-level `warnings`
  emitted only when non-empty. Two warnings fire: a config with no enabled output setting will write
  no files while `run_jobs` still reports `success:true`, and a `Quality`/`ConstantQP` rate control is
  named as unbounded below before the minutes are spent. **`minBitsPerPixel` judged OUT of scope, and
  the reason is the ticket's own thesis:** rewriting a caller's resolved config to `VariableBitRate` at
  a bitrate PinWright derived from a floor would make the plugin the party applying settings nobody
  asked for — this defect inverted — and disclosure plus the warning already gives a caller what they
  need to change one property on the preset asset. **Tests, behavioural, at the dispatcher.**
  `Tests/Media/TestMRQHandlers.cpp` gains `PinWright.mrq.create_job.UnloadablePresetIsNotReportedAsApplied`
  (the regression: fails against the old handler four independent ways — it answered success, set no
  error code, echoed the preset it could not load, and left a job in the shared queue) and
  `...PreflightDisclosesTheResolvedConfig`, which plants 1234x567 and stamped path formats on an
  in-memory preset so a block publishing constants or the engine CDO fails on the VALUES, not on their
  presence. `Tests/Media/TestMRQArtifactReport.cpp` gains
  `PinWright.mrq.create_job.PreflightWarnsUnboundedRateControl`, scoring the rate-control warning in
  BOTH directions — the engine's own `UMoviePipelineMP4EncoderOutput` CDO must warn, a
  `VariableBitRate` read-back must not — so a builder that warned unconditionally fails one half. Both
  handler tests restore the editor-global queue length even on their failing paths. **Stated
  limitations:** no MRQ render is driven (still PIE + minutes), so this exercises the job-configuration
  surface only; the in-memory-preset fixture emits `PINWRIGHT_ASSERTIONS_SKIPPED` rather than a red if a
  host cannot resolve it; and `Docs/error-code-catalog.md` was deliberately NOT hand-patched — it is a
  dated generated snapshot that instructs re-running its own scan. `Docs/wiki-src/mrq.md` documents the
  pre-flight shape, the refusal and the out-of-scope call. Not compiled and not run — a full automation
  suite was live against the compiled DLL for the whole of this work.
