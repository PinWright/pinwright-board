---
id: B-mrq-run-jobs-succeeds-on-unrenderable-frames
title: "mrq.run_jobs reports jobSucceeded:true with four healthy-looking 8.8 MB PNGs whose lower 47% is a flat white void and whose Nanite geometry has shattered — every field in the response is a file fact, and none of them is a picture fact"
status: OPEN
severity: High
category: bug
tags: [mrq, run_jobs, silent-false-success, movie-render-queue, nanite, lumen, warm-up, image-stats, acceptance-render]
encounters: 1
lastSeen: 2026-09-03T03:50:00Z
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
