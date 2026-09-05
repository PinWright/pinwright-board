---
id: B-mrq-run-jobs-succeeds-on-unrenderable-frames
title: "mrq.run_jobs reports jobSucceeded:true with four healthy-looking 8.8 MB PNGs whose lower 47% is a flat white void and whose Nanite geometry has shattered — every field in the response is a file fact, and none of them is a picture fact"
status: OPEN
severity: High
category: bug
tags: [mrq, run_jobs, silent-false-success, movie-render-queue, nanite, lumen, warm-up, image-stats, acceptance-render]
encounters: 2
lastSeen: 2026-09-05T18:19:00Z
---

# A render that cannot be used comes back as an unqualified success

`mrq.run_jobs` returned, verbatim:

    "jobSucceeded": true,
    "outputFileCount": 4, "measuredFileCount": 4,
    "totalFileSizeBytes": 35375075,
    "resolution": {"width": 3840, "height": 2160},
    "bitsPerPixel": 8.529869550540123,
    outputFiles: [ _0000.png exists:true 8839544, _0001.png exists:true 8829950,
                   _0002.png exists:true 8888151, _0003.png exists:true 8817430 ]

No `warnings`. Four files, all `exists:true`, all within 0.8 % of each other in size, 8.53
bits/pixel — comfortably above the response's own 0.04 bits/pixel plausibility floor.

**All four frames are unusable.** Opened and looked at: everything below roughly 53 % of frame
height is a **flat, uniform, pale grey-white field** with no geometry, no ground, no shadow and no
gradient — the yard the camera is pointed at is simply not there. Above that line the sky, the
perimeter wall, a container and the office block render correctly, so this is not a black frame, not
a blank frame, and not an exposure error. In frame `_0000` the container's corrugation has
additionally **shattered into radial spikes** and a wall panel is a sheared quad, i.e. geometry is
deforming as well as vanishing.

## What I called

    call({method: "mrq.create_job", args: {
      sequencePath: "/Game/FPS/Env/Cine/LS_ENV_Hero",
      levelPath:    "/Game/FPS/Maps/FPS_Compound",
      presetPath:   "/Game/FPS/Env/Cine/MPC_ENV_Hero_4K",
      jobName:      "ENVcritic_R2_4K" }})
    call({method: "mrq.run_jobs", args: {}})

`create_job`'s `preflight` was clean and accurate: 3840x2160, PNG image sequence, and the one
warning it raised was the correct "no video output setting is enabled" note. The preset carries
`EngineWarmUpCount` and `RenderWarmUpCount` (32 and 8, set by the ENV builder in the previous
session precisely to fix this) plus `SpatialSampleCount` / `TemporalSampleCount`. **The warm-up
counts did not fix it**, which is what makes the silent success expensive: the operator's only
feedback loop is the response, the response says success, and the failure is now two builds old.

## What I expected

Not a fix for the underlying void — that may well be Lumen/Nanite streaming and outside this verb.
What I expected is that **`jobSucceeded` should not be the only verdict on offer when every quantity
behind it is a property of the file rather than of the image.** `outputFiles`, `fileSizeBytes`,
`bitrate` and `bitsPerPixel` cannot distinguish a rendered frame from a frame that is half a solid
colour: a large flat region compresses well, but 3840x2160 of PNG keeps the file plausible, which is
exactly what happened here.

The sibling capture verbs already solve this. `render.capture_open_level` returns an `imageStats`
block (`meanLuminance`, `luminanceVariance`, `minLuminance`, `maxLuminance`, `toneLevelsUsed`,
`litPixelFraction`) plus `blank`, `crushed` and `blownOut` verdicts, and it refuses a near-uniform
frame outright with `BLANK_CAPTURE` unless `allowBlank:true`. A frame with 47 % of its pixels inside
a few luminance levels of each other would have been caught by `toneLevelsUsed` or by a
largest-uniform-region measure. `mrq.run_jobs` writes PNGs to disk and then measures only their size.

Asked for, in preference order:

1. Run the existing `imageStats` measurement over each written frame (or a subsample) and publish it
   per `outputFiles[]` entry, with the same `blank` / `crushed` / `blownOut` verdicts.
2. Failing that, one cheap uniformity statistic per frame — largest single-luminance region as a
   fraction of the frame — and a `warnings` entry when it exceeds, say, 25 %.
3. At minimum, stop letting `jobSucceeded:true` stand alone: rename it, or qualify it in the doc as
   "the pipeline ran to completion and wrote its files", which is all it currently means.

## Root cause guess

Not read from source. The `jobs[]` block is documented as measured off the finished files, and every
observed field is `stat()`-derived, so the likely shape is that the handler never opens the image
data at all. The void itself is probably a first-frame convergence state (the ENV stream's own note
records the same lower-half void with `engine_warm_up_count: 0`), but it now reproduces **with** 32
engine + 8 render warm-up frames, so warm-up count is not the whole mechanism.

## Workaround used

Open every rendered frame and look at it. There is no programmatic signal to gate on, so an
automated acceptance pipeline built on `jobSucceeded` would have shipped these four frames.

## Distinctness (dedup)

Searched the board for `mrq` (7 hits) and for the silent-success pattern. Not a duplicate of:

- `B-mrq-render-result-omits-bitrate-and-size` — asks for size/bitrate fields to exist at all; those
  fields are present here and are precisely the ones that mislead.
- `B-mrq-artifact-report-omits-stream-count` / `B-mrq-config-readback-omits-sampling` — missing
  metadata about the encode and the sampling config; this is about the absence of any measurement of
  the rendered pixels.
- `E-mrq-run-jobs-doc-promises-per-job-exit-status` — doc-vs-response drift on exit status;
  adjacent, but that ticket is about a field the doc names and the response omits, whereas
  `jobSucceeded` here is present and is affirmatively wrong as an acceptance signal.
- `B-capture-verbs-silent-default-material-fallback` — the same *class* (a capture verb reporting
  success and healthy stats over a wrong picture) on `asset.generate_thumbnail` /
  `render.capture_asset_preview`; different namespace, different mechanism, and that one at least
  published `imageStats` for two frames to be compared by. Cross-referenced, not merged.

## Fix

**Confirmed TRUE against source before changing anything.** `PinWrightMRQ::BuildArtifactReport`
(`Source/PinWright/Private/Handlers/MRQ/MRQArtifactReport.cpp`) opened no image data at all: its
only per-file measurement was `IFileManager::GetStatData` at the old `:144-158`, feeding `exists`,
`fileSizeBytes`, `totalFileSizeBytes` and everything derived from them. The root-cause guess in
this ticket was right.

Ask 1 was implemented (per-frame `imageStats` with the shipped verdicts), plus the statistic ask 2
called for, plus ask 3 in the doc. Two things the reporter did not ask for were added because they
are the same defect one layer up: the executor's non-fatal errors and the shots' pipeline state.

**New: the spatial statistic.** `Handlers/Render/FlatRegionStats.h/.cpp`
(`PinWrightFlatRegion::MeasureLargestFlatRegion`) — largest 4-connected region of one flat colour,
as a share of the frame, with its bounding box. It lives beside `SubjectRegionStats.h` rather than
inside `Handlers/MRQ/` because it is a frame property, not an MRQ property: `render.capture_*` has
the identical blind spot. **Why a new statistic and not a widened `blank`:** on a half-void frame
every whole-frame aggregate is healthy AND CORRECT — the good half carries a normal mean, a normal
variance and dozens of tone levels — so no threshold on them separates this from an ordinary
picture. Blocks are counted per AXIS (64), so the verdict is resolution-invariant the way
`BlankMinLitFraction` was rewritten to be. Regions grow anchored to their SEED block's level, never
to the frontier, which is what stops a sky gradient chaining into one giant "flat" region.

**New: the per-frame evidence.** `Handlers/MRQ/MRQFrameEvidence.h/.cpp` decodes a frame through the
existing `PinWrightImage::LoadBitmap`, measures it with the existing
`PinWrightRenderCapture::CalculateCaptureImageStats` (so `blank` / `crushed` / `blownOut` /
`toneLevelsUsed` mean here exactly what they mean on `render.capture_open_level`) plus the flat
region, and publishes `imageAnalyzed`, `imageStats`, the three verdicts, `suspect` and
`suspectReasons[]` (`FLAT_REGION` / `BLANK` / `CRUSHED` / `BLOWN_OUT`) on each `outputFiles[]`
entry, with `framesAnalyzed` / `framesSuspect` per job. **Sampled at 8 frames per job** (evenly
spaced, first and last always included) because decoding a 4K PNG is ~100 ms on the game thread and
a render can write hundreds; a warning fires whenever fewer frames were opened than were written,
so `framesSuspect: 0` can never be read as "every frame was checked". **Warns, never refuses** —
unlike `render.capture_open_level`'s `BLANK_CAPTURE`, because the files are already on disk and
refusing to report them would destroy the only record of what was written.

**The one causal thing it can honestly say.** It does NOT name the cause: an unconverged Nanite/VSM
stream, geometry that never loaded into the PIE world, a GPU timeout mid-accumulation and a matte
backdrop write the same pixels, and no readback separates them — so the warning lists them as
candidates and points at `shots[].state` and the queued level. What it DOES compute is where in the
render the suspect frames fall: all measured frames suspect ⇒ *not* first-frame convergence and
warm-up will not fix it; only the earliest ⇒ it is. That is exactly the conclusion this ticket
records taking two builds to reach by hand.

**Also surfaced, same defect class.** `mrq.run_jobs` now binds
`UMoviePipelineExecutorBase::OnExecutorErrored` and publishes `executorErrors[]`
(`fatal`, `message`, `jobName`). This is not redundant with `success`: `OnExecutorFinishedImpl`
broadcasts `!bAnyJobHadFatalError` and that flag is set only on the fatal branch, so a **non-fatal**
executor error left `success: true` and reached the caller nowhere — `executorWarning` now names
that combination. Each job also carries `shots[]` (`name`, `state` from `ShotInfo.State`) and warns
when a shot is not `Finished`. `jobSucceeded` is kept (renaming it breaks every existing caller) but
is documented in the verb description and the wiki as "the pipeline ran to completion and wrote its
files", which is all it ever meant.

Deliberately NOT done here: the anti-aliasing / warm-up / sampling config read-back. That is
`B-mrq-config-readback-omits-sampling`, still OPEN, and implementing it here would clobber its
scope.

### Files changed

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Render/FlatRegionStats.h` (new)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Render/FlatRegionStats.cpp` (new)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/MRQ/MRQFrameEvidence.h` (new)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/MRQ/MRQFrameEvidence.cpp` (new)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/MRQ/MRQArtifactReport.cpp` — frame sampling
  + analysis inside `BuildArtifactReport`, `framesAnalyzed` / `framesSuspect`, frame warnings
  emitted ahead of the encode warnings
- `Plugins/PinWright/Source/PinWright/Private/Handlers/MRQ/MRQArtifactReport.h` — header contract
- `Plugins/PinWright/Source/PinWright/Private/Handlers/MRQ/MRQHandler.cpp` — `OnExecutorErrored`
  binding, `executorErrors` / `executorWarning`, per-shot `shots[]` + unfinished-shot warning,
  rewritten `mrq.run_jobs` description
- `Plugins/PinWright/Source/PinWright/Private/Tests/Media/TestMRQArtifactReport.cpp` — 3 tests
- `Plugins/PinWright/docs/wiki-src/mrq.md` — two new `##` sections (kept above any `###`, page is
  13.4 KB, under the ~20 KB guideline)

### Tests added

- `PinWright.mrq.run_jobs.FrameVoidIsSuspect` — writes two REAL 256x256 PNGs that differ only in
  whether the lower half is one flat colour, runs both through the shipped `BuildArtifactReport`,
  and asserts the void frame comes back `suspect: true` / `FLAT_REGION` with
  `flatRegionFraction ≈ 0.5` and `flatRegionBounds.minY ≈ 0.5`, while `blank` / `crushed` /
  `blownOut` stay FALSE and `fileSizeBytes` stays healthy — i.e. the ticket's misleading evidence is
  reproduced in the same response as the correct verdict. The textured control frame must come back
  clean, so a builder that flags everything fails.
- `PinWright.mrq.run_jobs.NonImageArtifactIsNotReportedClean` — an `.mp4` gets
  `imageAnalyzed: false` present-and-false with a reason, and no invented `suspect`.
- `PinWright.mrq.run_jobs.FlatRegionDoesNotChainAGradient` — a vertical ramp of one level per block
  row: every block is individually flat, so a frontier-chaining implementation would report the
  whole frame as one void; the shipped seed anchor caps it under 15 %. A uniform frame in the same
  test must still report >99 %, so the first half cannot pass by never finding anything.

Not compiled and not run — a separate compile pass follows.

### Reviewer verification

1. Compile; run `PinWright.mrq.*` (3 new tests, 4 existing MRQ tests, all offline — no editor
   render needed).
2. Live: `mrq.create_job` + `mrq.run_jobs` on any sequence writing a PNG sequence. Confirm each
   `outputFiles[]` entry carries `imageAnalyzed`, and the opened ones carry `imageStats` with
   `flatRegionFraction` / `flatRegionBounds`, `blank` / `crushed` / `blownOut`, `suspect`; confirm
   `framesAnalyzed` / `framesSuspect` at job level and `shots[].state`.
3. The ticket's own repro (`LS_ENV_Hero` + `MPC_ENV_Hero_4K` over `FPS_Compound`) should now come
   back `framesSuspect > 0` with a `flatRegionBounds.minY ≈ 0.53` and the "EVERY frame that was
   measured is affected" note. If the void no longer reproduces, verify instead against any render
   whose frame carries a large flat sky and check the region bounds name the TOP of the frame — the
   warning is expected to fire there too, and that false positive is by design.
4. Cost check: on a render writing >8 still frames, confirm only 8 are decoded and that the
   sampling warning is present.

## History
- `#1-filed` `OPEN` reporter — Filed from the ENV round-2 blind-A/B critic pass. `mrq.run_jobs` on
  `LS_ENV_Hero` + `MPC_ENV_Hero_4K` over `/Game/FPS/Maps/FPS_Compound` returned `jobSucceeded:true`,
  4 files, `totalFileSizeBytes` 35,375,075, `bitsPerPixel` 8.53, no `warnings`. All four 3840x2160
  frames carry a flat pale-grey void below ~53 % of frame height where the compound yard should be,
  and `_0000` additionally shows the container's corrugation exploded into radial spikes and a
  sheared wall panel. The preset already carried the 32-engine + 8-render warm-up counts added
  specifically to fix this in the prior session, so warm-up is not the mechanism and the silent
  success has now survived two builds. Every field the response offers is a file property; none
  looks at the pixels, while the sibling `render.capture_open_level` already computes `imageStats`
  plus `blank`/`crushed`/`blownOut` and refuses a near-uniform frame with `BLANK_CAPTURE`. Asked for
  per-frame `imageStats` on `outputFiles[]`, or failing that a largest-uniform-region fraction with
  a `warnings` entry past ~25 %, or at minimum a doc/name change so `jobSucceeded` reads as "the
  pipeline completed and wrote files" rather than as an acceptance verdict. Workaround: open and
  look at every frame; there is no signal to gate on.
- `#2-frame-evidence-implemented` `IN-REVIEW` developer — Verified TRUE by reading source:
  `BuildArtifactReport` measured files only, never pixels. `mrq.run_jobs` now DECODES a bounded
  sample of the written frames (8 per job, evenly spaced, first and last always included) and
  publishes per `outputFiles[]` entry `imageAnalyzed`, `imageStats` (the shipped
  `CalculateCaptureImageStats` fields plus a new `flatRegionFraction` / `flatRegionLevel` /
  `flatRegionBounds` / `flatBlockFraction`), `blank` / `crushed` / `blownOut`, `suspect` and
  `suspectReasons[]`, rolled up as `framesAnalyzed` / `framesSuspect`. The new statistic
  (`Handlers/Render/FlatRegionStats.*`) is the largest 4-connected flat region, block-partitioned
  per axis so it is resolution-invariant and seed-anchored so a sky gradient cannot chain into one
  region — a whole-frame aggregate structurally cannot see a half-void frame. It warns and never
  refuses, and it does not claim the cause; it does report WHERE in the render the suspect frames
  fall, which separates "unconverged start, add warm-up" from "warm-up will not fix this". Also
  added: `executorErrors[]` from `OnExecutorErrored` (a NON-fatal executor error leaves
  `success:true` and previously reached the caller nowhere) with `executorWarning`, and per-job
  `shots[]` with a warning for any shot not `Finished`. `jobSucceeded` kept but documented as
  "the pipeline ran to completion and wrote its files". Three tests added; not compiled here — a
  separate compile pass follows. The sampling-config read-back was deliberately left to
  `B-mrq-config-readback-omits-sampling`. See the Fix section for files and verification steps.
- `#2-returned` `OPEN` ENV — **Returned to OPEN: the new pixel analysis does not catch the frame this ticket was filed for.** UE 5.8, editor PID after the 18:10:30Z restart, `/Game/FPS/Env/Cine/LS_ENV_Hero` + `MPC_ENV_Hero_4K` (now carrying the 32+8 warm-up fix), `/Game/FPS/Maps/FPS_Compound`. All four 3840x2160 frames still have a **flat void filling the lower ~47 percent** - the identical defect - and the verb passed every one of them:

  ```
  framesAnalyzed 4   framesSuspect 0   jobSucceeded true   shots[0].state Finished
  frame 0003: suspect false   blank false   crushed false   blownOut false
              meanLuminance 0.6529   luminanceVariance 0.0611   toneLevelsUsed 256
              flatRegionFraction 0.005859   flatRegionLevel 3
              flatRegionBounds { minX 0.953, minY 0.1875, maxX 1.0, maxY 0.375 }
  ```

  `flatRegionBounds` locates a sliver **at the right edge**, 4.7 percent of the width and a fifth of the height. The actual void spans the full width and the bottom 47 percent, and the detector scored it at 0.6 percent.

  **Why it is missed, which is the actionable part.** The void is not a constant-valued block - it is a very shallow vertical *gradient* of pale blue-grey. A detector keyed on runs of one quantised level (`flatRegionLevel` is a single level, 2 or 3 across the four frames) walks off the region as soon as the level steps, so a gradient of a few levels over 1000 px never accumulates. The global stats are no help either and look actively healthy: `toneLevelsUsed` is a full 256 and variance 0.061, both earned by the intact top half. So every published field agrees the frame is fine.

  **What would catch it**, offered as the same class of measure the verb already computes: low LOCAL variance over a large connected area, not equality of level. A block-wise variance map at, say, 32x32 with a threshold on the largest connected low-variance component would score this frame near 0.47. `flatBlockFraction` is already reported (0.034 here) and looks like the right quantity measured at the wrong scale or threshold.

  Not re-filed as a new ticket because it is this ticket's own acceptance criterion - a frame whose lower half is a void must not come back `suspect: false`. **Workaround back in force:** the hero frame is captured with `render.capture_open_level` at 3840x2160 instead (litPixelFraction 0.9967, toneLevelsUsed 256, verified by eye), and MRQ output is not trusted without opening the file. `encounters` 1 -> 2.
