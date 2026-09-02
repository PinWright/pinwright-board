---
id: B-mrq-config-readback-omits-sampling
title: "The mrq preflight discloses the encode and nothing else: the anti-aliasing sample counts and warm-up frame counts that decide whether the frames converged are read by no verb, and `outputs[]` lists only file writers, so a caller cannot even enumerate what settings the config carries"
status: OPEN
severity: Medium
category: bug
tags: [mrq, create_job, run_jobs, movie-pipeline, preflight, readback, anti-aliasing, warm-up, sampling, config-disclosure, response-honesty]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# The report answers "how was it encoded" and cannot answer "how was it sampled"

`mrq.create_job`'s `preflight` block and `mrq.run_jobs`' per-job report both read the job's
*resolved* configuration, which is the right source. Both read exactly two things off it: the
output setting, and the first video output setting. Everything else the configuration carries —
including the settings that decide whether the pixels are converged — is invisible to every verb
in the namespace.

## What is read, exhaustively

`ReadPreflightContext` (`Source/PinWright/Private/Handlers/MRQ/MRQHandler.cpp:155`) touches the
config in three places and no others:

- `Config->FindSetting<UMoviePipelineOutputSetting>()` (`:167-181`) → resolution, output directory,
  filename format, and a frame-rate override when `bUseCustomFrameRate` is set;
- a loop over `Config->FindSettingsByClass(UMoviePipelineOutputBase::StaticClass())` (`:184-192`)
  → `Context.OutputClassPaths`, published as `preflight.outputs`;
- `Config->FindSettingsByClass(UMoviePipelineVideoOutputBase::StaticClass())` (`:193-198`) → the
  `encoderRequested` reflection read-back.

`MakeJobArtifactReport` reads the same two classes on the finished render
(`MRQHandler.cpp:117-136`). Nothing anywhere reads a third.

**So `preflight.outputs` is not a list of the config's settings — it is a list of its enabled *file
writers*.** The wiki says exactly that (`Saved/PinWright/wiki/mrq.md:41`: *"class path of every
**enabled** file writer on the config"*), and it is easy to read as a config inventory when it is
the one array-of-classes the response carries. A config with an anti-aliasing setting, a camera
setting, console-variable overrides, game overrides and a high-res tiling setting reports exactly
the same `outputs[]` as one with none of them.

## What is never read

A tree-wide, case-insensitive grep over `Source/` for
`MoviePipelineAntiAliasingSetting|SpatialSampleCount|TemporalSampleCount|EngineWarmUpCount|RenderWarmUpCount`
returns **zero hits**, and the same grep for `antialias|spatialsample|temporalsample|warmup|samplecount`
scoped to `Source/PinWright/Private/Handlers/MRQ/` (all three files) returns **zero hits**. The
sampling half of an MRQ configuration is not settable, not readable, and not mentioned on any of
the four `mrq` wiki pages.

That is the half that decides whether a frame is finished. `UMoviePipelineAntiAliasingSetting`
carries the spatial and temporal sample counts and the engine/render warm-up frame counts; a
render at 1 temporal sample with no warm-up produces frames that are noisy, unconverged and
missing their first-frame ambient contribution — while resolution, frame count, duration, file
size, bitrate and `encoderRequested` all come back exactly correct. That is the same failure shape
`B-mrq-render-result-omits-bitrate-and-size` was filed for, one setting class over.

## Encountered, 2026-09-02

Producing the Atlantis showcase video required these settings to be right, and they are recorded
in the project's own build note because no verb could report them
(`Docs/map/atlantis-video-plan.md:126-129`):

    Deferred pass, PNG, TSR, **4 temporal / 1 spatial sample, engine warm-up 300**,
    `use_camera_cut_for_warm_up=false`; game overrides `view_distance_scale 50`,
    `shadow_distance_scale 10`, LOD 0, texture streaming off, grass overrides all off.

Every value in that sentence had to be authored and then re-read outside the `mrq` namespace. Of
the eight per-tile presets (`/Game/Atlantis/Cine/Video/MPC_Vid_*`), the only fact `mrq.create_job`
would disclose is resolution, output directory, filename format and the PNG writer's class path.

The two flythrough renders are the sharper case: `Content/Atlantis/Cine/` holds
`LS_Atlantis_Flythrough.uasset` and **no `MPC_*` asset beside it**, so both were queued with no
`presetPath` at all and ran on the engine's CDO defaults. `mrq.create_job` now warns about the
rate control in that situation (`MRQArtifactReport.cpp:356-369`) — correctly, and that is the
landed half of the sibling ticket — but it says nothing about the sample counts and warm-up the
same CDO also chose.

## Ask

Extend the same read-off-the-resolved-config pattern by one setting class and one array:

- **`preflight.sampling`** — read `UMoviePipelineAntiAliasingSetting` by the reflection idiom the
  encoder read-back already uses (`ReadRequestedEncoderSettings`,
  `Handlers/MRQ/MRQArtifactReport.h:84-88`, chosen so it costs no module dependency and works on
  any class carrying the same-named properties): spatial sample count, temporal sample count,
  engine warm-up count, render warm-up count, and the anti-aliasing method override.
  **Absent when the config carries no such setting**, matching the report's existing
  omit-rather-than-zero contract (`MRQArtifactReport.h:31-35`) — "the config has no AA setting" is
  itself the answer a caller needs, and it is a different answer from "1 sample".
- **`preflight.settings[]`** — the class path of *every* setting on the resolved config, not only
  the file writers, so `outputs[]` stops being the only inventory a caller can see and stops being
  mistakable for one. Cheap: it is the same `FindSettingsByClass` loop against
  `UMoviePipelineSetting::StaticClass()`.
- Mirror both into `mrq.run_jobs`' per-job report, which already re-reads the config for
  resolution and the encoder, so the finished render's own account names how it was sampled.

Deliberately **not** asked: that PinWright *set* any of these. The namespace's standing position
is that it discloses and never applies settings nobody asked for
(`Saved/PinWright/wiki/mrq.md:47`, and `MRQHandler.cpp:126-127`), and this ticket agrees with it —
disclosure is precisely what is missing.

## Related

- `B-mrq-render-result-omits-bitrate-and-size` (High, IN-REVIEW) — the ticket whose landed work
  built `preflight` and the artifact report. Its Ask 2 is "report the configuration a render used",
  and it delivered that for the encode. This is the same ask for the sampling. Filed as a sibling
  rather than appended as an encounter because that ticket is IN-REVIEW awaiting a tester and a new
  omission folded into it would blur what is being verified.
- `B-mrq-artifact-report-omits-stream-count` — the same pair of reports' blind spot on the output
  side: what the produced file contains.
- `F-mrq-preset-authoring` — why the values had to be authored out of band in the first place.

## History
- `#1-sampling-settings-invisible` `OPEN` reporter — "Filed 2026-09-02 from the Atlantis showcase video. `ReadPreflightContext` (`MRQHandler.cpp:155`) reads exactly `UMoviePipelineOutputSetting` (`:167-181`), the enabled `UMoviePipelineOutputBase` list (`:184-192`) and the first `UMoviePipelineVideoOutputBase` (`:193-198`); the run-time reader does the same two (`:117-136`). Tree-wide grep for `MoviePipelineAntiAliasingSetting|SpatialSampleCount|TemporalSampleCount|EngineWarmUpCount|RenderWarmUpCount` across `Source/` returns zero hits, and the four `mrq` wiki pages never mention sampling or warm-up. So `preflight.outputs` is a file-writer list (`wiki/mrq.md:41`), not a settings inventory, and the sample counts that decide convergence are unreadable. Encountered producing the video: the tiles needed 4 temporal / 1 spatial / engine warm-up 300 (`Docs/map/atlantis-video-plan.md:126-129`), authored and verified entirely outside the namespace, and neither flythrough render had a preset asset at all (`Content/Atlantis/Cine/` holds the sequence and no `MPC_*`), so their sampling could only be inferred."
