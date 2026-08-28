---
id: B-thumbnail-cold-first-frame-no-stats
title: "The first `asset.generate_thumbnail` after a compile returns a blank or half-drawn frame with a success payload identical to the good one — the verb publishes NO image statistics, so a cold frame is indistinguishable from a real one and fakes a motion delta against a warm one"
status: DONE
severity: High
category: bug
tags: [asset, generate_thumbnail, thumbnail, cold-frame, warmup, false-evidence, no-image-stats, silent-false-success, visual-verification, motion-proof, readback]
encounters: 1
lastSeen: 2026-08-28
---

# A cold frame and a finished frame return the same success payload, and the caller has no field to tell them apart

The first thumbnail taken of a material after `material.authoring.compile_material` comes back
blank, or drawn with an unfinished shader. A second, identical call renders it correctly. Nothing
in either response distinguishes them: both return `success: true` with the same `width`, `height`,
`format` and `outputPath`.

The measured signatures are unambiguous once you go looking. The problem is that
`asset.generate_thumbnail` gives a caller nothing to look at, so you only go looking after
something downstream has already gone wrong.

## The correction the obvious hypothesis needs — this is NOT a compilation-state bug

The natural fix to reach for is "flush shader compilation before the readback, the way
`material.authoring.compile_material` already blocks on `GShaderCompilingManager`". That aim is
wrong, and the board already contains the reason:

- `B-compile-material-false-shader-success` (DONE, High) `#2` made `compile_material` block, via a
  new `MaterialCompileErrorCollector::WaitAndCollect`.
- `B-compile-material-blocks-and-mislabels` (OPEN, High) confirms it still does — and its whole
  complaint is that the block is *too* thorough (a ~4-minute synchronous game-thread stall with no
  job handle).

Both citations re-verified against this tree at HEAD, with one line-number drift worth recording:
that ticket cites `MaterialAuthoringHandler.cpp:3476`; the call is now at
**`MaterialAuthoringHandler.cpp:3487`**:

```cpp
    MaterialCompileErrorCollector::WaitAndCollect(Material, CompileErrors);
```

and `MaterialCompileErrorCollector.h:46,51` is unchanged:

```cpp
        Material->CacheShaders(EMaterialShaderPrecompileMode::Synchronous);   // :46
        if (GShaderCompilingManager)
        {
            GShaderCompilingManager->FinishAllCompilation();                  // :51
        }
```

So the compile genuinely completes before the thumbnail is requested. **The coldness is in
`asset.generate_thumbnail`'s own render/readback path, not in compilation state**, and a fixer who
adds a second shader flush there will find it changes nothing.

## The primary ask: publish image statistics, the way the `render.*` verbs already do

`asset.generate_thumbnail` reports **no image statistics at all**. Its entire success payload,
verbatim from `Handlers/Asset/AssetWorkflowHandler.cpp:835-862`, is `success`, `assetPath`,
`width`, `height`, optional `requestedWidth` / `requestedHeight`, optional `primitive`, and — only
when `outputPath` was given — `outputPath` and `format`. There is no mean, no variance, no blank
flag, no tone-range verdict. A caller cannot distinguish a cold frame from a real one without
diffing it against another capture.

**The machinery that would have caught this instantly is already written and shipping, one
namespace over.** `B-exposure-pin-black-frame` (IN-REVIEW, High) `#4` landed a degenerate-frame
classifier for the three `render.*` single-frame verbs: it counts how many of 256 8-bit luminance
levels the pixels populate, calls fewer than eight a collapse, and publishes `crushed` / `blownOut`
top-level beside `blank`, with `toneLevelsUsed` and `toneLevelMinPixels` in `imageStats` and a
`rangeWarning` string quoting the measured count, the mean and the direction. An empty portal frame
and a washed-out grey cube are exactly what that classifier exists to name.

That is the ask, and it is the one that generalises: **give `asset.generate_thumbnail` the same
`imageStats` block and the same degenerate-frame verdict.** Reusing the shipped classifier rather
than writing a second one also avoids the drift that `B-exposure-pin-black-frame` `#4` had to fix
in its own follow-up commit.

A warm-up frame or a pre-wait is the secondary ask. It stops the bad frame; only the statistics let
a caller *know* they got one — including on the failure modes nobody has hit yet.

## Root cause — NOT traced to the observed frames; two candidates verified present in source

Stated as candidates on purpose. Nobody has bisected this to a guilty line, and no A/B was run in
this filing pass. What follows is what is verifiably in the code, not a claim about which one
produced the two frames below.

**Candidate 1 — the handler renders once with texture streaming explicitly not flushed.**
`Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp:779-782`:

```cpp
    ThumbnailTools::RenderThumbnail(
        Asset, Width, Height,
        ThumbnailTools::EThumbnailTextureFlushMode::NeverFlush, nullptr,
        &ObjectThumbnail);
```

One call, no pump loop, no warm-up, no retry. `NeverFlush` is documented in
`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Public/ObjectTools.h:766-777` as "Don't flush texture
streaming at all", against `AlwaysFlush` — "Aggressively stream resources before rendering
thumbnail to avoid blurry textures". Choosing `NeverFlush` skips, at
`ObjectTools.cpp:5730-5743`, all four of `FlushAsyncLoading()`,
`FAssetCompilingManager::Get().FinishAllCompilation()`, `UTexture::ForceUpdateTextureStreaming()`
and `IStreamingManager::Get().StreamAllResources()`. The washed-out semi-transparent grey cube
below is what unstreamed mips look like.

**Candidate 2 — the engine's own thumbnail generator does per-class pre-waits that this handler
does not.** `ObjectTools::GenerateThumbnailForObjectToSaveToDisk` (`ObjectTools.cpp:5870-5912`)
also passes `NeverFlush` (`:5883`), and compensates first:

```cpp
			if ( UTexture* Texture = Cast<UTexture>(InObject) )
			{
				Texture->BlockOnAnyAsyncBuild();                              // :5888
				Texture->WaitForStreaming();                                  // :5889
			}
			// When generating a material thumbnail to save in a package, make sure we finish compilation on the material first
			if ( UMaterial* InMaterial = Cast<UMaterial>(InObject) )
			{
				FMaterialResource* CurrentResource = InMaterial->GetMaterialResource(GMaxRHIShaderPlatform);
				if (CurrentResource)
				{
					if (!CurrentResource->IsGameThreadShaderMapComplete())    // :5902
					{
						CurrentResource->SubmitCompileJobs_GameThread(EShaderCompileJobPriority::High);
					}
					CurrentResource->FinishCompilation();                     // :5906
				}
			}
```

Note what that material branch is: a **game-thread shader-map completeness** check on the
`FMaterialResource`, which is not the same question `WaitAndCollect`'s
`GShaderCompilingManager->FinishAllCompilation()` answers, and the engine comment says it is
required specifically for thumbnails. That is consistent with the correction above — compilation
finished, and the thumbnail path still has its own readiness precondition — but it is a hypothesis
about the mechanism, **not verified against the measured frames**.

**The control that would settle it:** re-run the two-call sequence with the handler patched to
`AlwaysFlush`, and separately with the two per-class pre-waits added, and see which of the two
frames stops being cold. Neither was run here.

## Verbatim repro and measured signatures

Two identical calls in a row, minutes apart, on a freshly compiled material (2026-08-27, UE 5.8,
this checkout):

```
material.authoring.compile_material {assetPath:"/Game/Atlantis/Materials/M_PortalGlow"}
asset.generate_thumbnail {assetPath:"/Game/Atlantis/Materials/M_PortalGlow",
                          primitive:"plane", width:512, height:512, outputPath:"t0.png"}
asset.generate_thumbnail {... identical ..., outputPath:"t1.png"}     // later
```

Measured with `image.compare` this session:

- `scratchpad/cmp_portal_side_by_side.png` — A (first call) is an **entirely empty frame**;
  B (second call) is the finished turquoise portal.
  `differingFraction 0.9964`, `meanAbsDifferenceOverall 47.66`,
  `bestFitGain {r:0.057, g:0.046, b:0.046}`.
- `scratchpad/cmp_water_side_by_side.png` — A (first call) is a **washed-out, semi-transparent
  grey** cube; B is the correct opaque cyan.
  `bestFitGain {r:1.463, g:0.615, b:0.549}` — the two frames do not even share a hue.

**The heuristic, worth putting in the docs regardless of the fix:** on a comparison pair, read
`bestFitGain` before believing anything. A gain near **0.05** means one side is near-black. A
**per-channel disagreement** (r/g/b gains that are not roughly equal) means the two frames do not
share a hue. Either one means one frame is not comparable to the other, and the measured difference
between them means nothing.

## The real cost — this fakes evidence in both directions

This is the failure mode the vision-verification rule exists to catch, and it defeats it:

1. A cold frame reads as **"the material is broken"** when the material is fine.
2. Worse for a two-instant animation check: a cold t0 against a warm t1 produces a large measured
   difference that has nothing to do with the material animating. **Two of the four motion proofs
   on this build were invalidated by it and had to be re-shot.** Without `bestFitGain` and the
   side-by-side picture, both would have been reported as proof of motion that they are not.

## Workaround

Take a throwaway capture first and judge only the second and later shots. This is exactly the rule
`render.capture-exposure` § *The first capture into a fresh preview window is a stop dark* already
states for `render.capture_asset_preview` — **that section does not mention
`asset.generate_thumbnail`**, and this is worse than a stop of exposure: it is a blank or
half-drawn frame. Add the verb to that section as part of the fix, whichever way the fix goes.

## Distinct from related tickets

- `B-exposure-pin-black-frame` (IN-REVIEW, High) is the **capability precedent, not a duplicate**:
  same class of harm (a degenerate frame reported as a clean success) on `render.capture_asset_preview`,
  fixed there by measuring the frame. Its `#4` machinery is what this ticket asks to be extended to
  a fourth verb. Different handler, different cause — that one is an exposure pin crushing a frame
  the renderer drew correctly; this one is a frame that was not finished being drawn.
- `B-compile-material-false-shader-success` (DONE) and `B-compile-material-blocks-and-mislabels`
  (OPEN) are the two tickets that between them prove `compile_material` *does* block, which is why
  the fix must not be aimed at compilation state. Cited as the correction, not as related work.
- `B-capture-asset-preview-renders-empty` (IN-REVIEW) is the non-realtime white-frame bug on a
  different verb and a different frame.
- `B-open-level-blank-success` (IN-REVIEW) is uniform-black classification on
  `render.capture_open_level` — the origin of the `blank` gate this verb also lacks.
- `B-thumbnail-png-writes-jpeg` (DONE) fixed this verb's encoder and its alpha and size reporting.
  It never touched frame readiness or frame measurement.

severity rationale: impact=silent false-success on a normal path — a blank or half-drawn frame returns a success payload byte-for-byte indistinguishable from a good one, and a caller building a comparison on it measures a difference that is entirely artefact (two motion proofs invalidated on this build) x reach=the only verb that renders a material at all (see `E-material-capture-no-thumbnail-pointer`), so every material visual check goes through it -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Two identical `asset.generate_thumbnail` calls minutes apart after `material.authoring.compile_material`: the first returns a blank or half-drawn frame, the second the correct one, with identical `success`/`width`/`height`/`format`/`outputPath`. Measured with `image.compare`: `/Game/Atlantis/Materials/M_PortalGlow` — `differingFraction 0.9964`, `meanAbsDifferenceOverall 47.66`, `bestFitGain {r:0.057,g:0.046,b:0.046}` (near-black first frame); the water material — `bestFitGain {r:1.463,g:0.615,b:0.549}` (frames do not share a hue). Heuristic recorded: a gain near 0.05, or per-channel disagreement, means one frame is not comparable. **Corrected framing, verified in this tree:** the log's proposed fix (flush shader compilation the way `compile_material` does) is aimed wrong — `compile_material` already blocks (`B-compile-material-false-shader-success` `#2` shipped it; `B-compile-material-blocks-and-mislabels` complains the block is a ~4-min game-thread stall), and both citations re-verified at HEAD with a line-number drift: the `MaterialCompileErrorCollector::WaitAndCollect` call is now `MaterialAuthoringHandler.cpp:3487`, not `:3476`; `MaterialCompileErrorCollector.h:46,51` unchanged (`CacheShaders(Synchronous)` + `GShaderCompilingManager->FinishAllCompilation()`). So the compile completes before the thumbnail is requested and the coldness is in the thumbnail render/readback path. **Root cause NOT traced to the observed frames**; two candidates verified present in source and explicitly unproven: (1) `AssetWorkflowHandler.cpp:779-782` renders once with `EThumbnailTextureFlushMode::NeverFlush`, which skips `FlushAsyncLoading` / `FAssetCompilingManager::FinishAllCompilation` / `UTexture::ForceUpdateTextureStreaming` / `IStreamingManager::StreamAllResources` (`ObjectTools.cpp:5730-5743`; the mode is documented at `ObjectTools.h:766-777` with `AlwaysFlush` described as avoiding blurry textures); (2) the engine's own `ObjectTools::GenerateThumbnailForObjectToSaveToDisk` (`ObjectTools.cpp:5870-5912`) passes the same `NeverFlush` but first does per-class pre-waits this handler omits — `BlockOnAnyAsyncBuild()`+`WaitForStreaming()` for `UTexture` (`:5888-5889`) and, for `UMaterial`, `SubmitCompileJobs_GameThread(High)` + `FinishCompilation()` when `!IsGameThreadShaderMapComplete()` (`:5902-5906`), a game-thread shader-map readiness check distinct from what `WaitAndCollect` answers. Control that would settle it (not run here): re-run the pair with `AlwaysFlush`, and separately with the two pre-waits, and see which frame stops being cold. **Primary ask is the reporting, not the warm-up**: `asset.generate_thumbnail` publishes no image statistics at all (`AssetWorkflowHandler.cpp:835-862` — success/assetPath/width/height/requestedWidth/requestedHeight/primitive/outputPath/format and nothing else), while `B-exposure-pin-black-frame` `#4` already shipped `crushed`/`blownOut`/`toneLevelsUsed`/`toneLevelMinPixels`/`rangeWarning` for the three `render.*` single-frame verbs — machinery that would have named both of these frames instantly. Cost: a cold frame reads as a broken material, and a cold t0 against a warm t1 fakes a large motion delta — **two of the four motion proofs on this build were invalidated and had to be re-shot**. Worked around by discarding the first capture, the rule `render.capture-exposure` already states for `render.capture_asset_preview` and which does not mention this verb; defect untouched.

- `#2-frame-evidence-and-subject-readiness` `IN-REVIEW` developer — Both halves shipped. (1) EVIDENCE: `asset.generate_thumbnail` now measures its own pixels on EVERY call (with or without `outputPath`) through the shipped classifier `PinWrightRenderCapture::CalculateCaptureImageStats` — no second criterion was written. New response fields: `imageStats` (`meanLuminance`/`luminanceVariance`/`minLuminance`/`maxLuminance`/`litPixelCount`/`litPixelFraction`/`litLuminanceThreshold`, plus `toneLevelsUsed`/`toneLevelMinPixels` via the same `AddToneRangeStatsFields` writer the `render.*` verbs use), top-level `blank`/`crushed`/`blownOut`, `imageStatsMeasured`, `frameWarning`, `renderPasses`, `coldFrameRetry`, `readiness`. `frameWarning` is thumbnail-specific rather than `MakeToneRangeWarning`, which names an `ev100` this verb does not have; the verdicts themselves keep the `render.*` meaning. Unmeasured pixels publish NO statistics (an omitted `toneLevelsUsed` rather than a 0 that would read as the most collapsed frame). (2) READINESS: new `WaitForThumbnailSubjectReadiness` runs the engine's own thumbnail pre-waits (`ObjectTools::GenerateThumbnailForObjectToSaveToDisk`) SCOPED TO THE SUBJECT rather than globally — `FAssetCompilingManager::FinishCompilationForObjects({asset, its used textures})`, `WaitForStreaming()` on every texture the frame samples (candidate 1's unstreamed mips), and the game-thread `IsGameThreadShaderMapComplete()` / `SubmitCompileJobs_GameThread(High)` / `FinishCompilation()` gate (candidate 2), widened from the engine's `UMaterial`-only branch to `UMaterialInterface` so instances get it too. `shaderMapCompleteBefore`/`After` are read back, not assumed, so a `false`→`true` pair is direct evidence the call would have rendered cold. No global `AlwaysFlush` (that is the ~4-min stall `B-compile-material-blocks-and-mislabels` complains about), and no fixed-count warm-up: the ONE extra render pass is gated on the measurement (`FrameIsDegenerate`) and bounded at one, with both passes' numbers plus the between-pass differing-pixel count published as `coldFrameRetry`. The wait sits above `FScopedPreviewOverride` so it cannot widen that guard's mutation window. Files: new `Handlers/Asset/ThumbnailFrameEvidence.h`/`.cpp`; `Handlers/Asset/AssetWorkflowHandler.cpp` (readiness call, render-pass lambda + measurement + gated retry, one `AddFrameEvidenceFields` line). Tests added in `Tests/Assets/TestGenerateThumbnail.cpp`: `PinWright.asset.generate_thumbnail.FrameEvidenceSeparatesColdFromReal` (synthetic all-black, flat-grey and full-range-ramp buffers; asserts the flat-grey frame is NOT blank yet is still flagged, that the cold and finished payloads now differ, that unmeasured pixels publish nothing, and the retry block's differing fraction) and `PinWright.asset.generate_thumbnail.HandlerPublishesMeasuredFrameEvidence` (drives the real verb, then RE-MEASURES the written PNG and requires the published `meanLuminance`/`maxLuminance`/`toneLevelsUsed`/`blank` to describe those bytes). No new `ERR_*` codes. No asset-dump aspect bump needed — `preview.png` is the widget-designer path (`WidgetDesignerCaptureUtil`), not this verb; `ThumbnailTools::RenderThumbnail` has exactly one call site in the tree. Not compiled or run: the orchestrator owns builds. `docs/wiki-src/asset.md` prose reported to the orchestrator, not edited here.

- `#3-verified-behaviourally-on-the-built-binary` `DONE` verifier — 2026-08-28, against the
  `b79ba53e` build. `#2` closed *"Not compiled or run"*. Both halves now measured live.

  **(1) The statistics are published on every call, and they describe the bytes that were written.**
  Every `asset.generate_thumbnail` response in this session carried `imageStats`
  (`meanLuminance`/`luminanceVariance`/`min`/`max`/`litPixelCount`/`litPixelFraction`/
  `litLuminanceThreshold`/`toneLevelsUsed`/`toneLevelMinPixels`), `blank`/`crushed`/`blownOut`,
  `imageStatsMeasured`, `renderPasses` and `readiness` — with and without `outputPath`. The claim
  was checked independently rather than trusted: three written PNGs were decoded with a standalone
  stdlib decoder and their Rec.709 luminance recomputed from the pixel bytes.

  | file | published mean / var / min / max / litFraction | **recomputed from the PNG bytes** |
  |---|---|---|
  | `MI_Fish_Silver` sphere | 0.5639504 / 0.1524380 / 0.0235294 / 0.9523906 / 1.0 | 0.563950 / 0.152438 / 0.023529 / 0.952391 / 1.000000 |
  | `DefaultMaterial` plane | 0.1269719 / 0.0183372 / 0.0196078 / 0.3692298 / 0.975 | 0.126972 / 0.018337 / 0.019608 / 0.369230 / 0.975000 |
  | 512x512 Perlin noise | 0.4057468 / 0.0275926 / 0.0313725 / 0.9372549 / 1.0 | 0.405747 / 0.027593 / 0.031373 / 0.937255 / 1.000000 |

  Agreement to six decimals on all three, `litPixelCount` exact (102400 / 99840 / 262144). The
  published numbers are a measurement of the delivered image, not of some earlier buffer.

  **(2) The readiness wait is real and its evidence is read back, not assumed.** `readiness` carried
  `shaderMapCompleteBefore: false` -> `shaderMapCompleteAfter: true` on the first call to
  `M_Fish`, on the first call to `/Engine/EngineMaterials/DefaultMaterial`, and on `M_Crosshair` —
  three calls that would have rendered cold and now do not. The immediately following second call to
  `DefaultMaterial` reported `shaderMapCompleteBefore: true`. That is precisely the cold/warm pair
  the ticket said no field could distinguish, and it now sits in the payload.

  **(3) The gated retry and the degenerate-frame verdicts fire, and separate "flat asset" from
  "cold frame".** A deliberately uniform texture (`create_noise_texture scale:1 seamless:true`,
  source min = max = 91) rendered with `renderPasses: 2` — the retry triggered on the measurement,
  bounded at one — and returned `crushed: true`, `blank: false`, `toneLevelsUsed: 1`,
  `luminanceVariance: 8.2e-14`, plus `coldFrameRetry` with `firstPassToneLevelsUsed: 1`,
  `differingPixels: 0` of `comparedPixels: 16384`, `differingFraction: 0`, `frameChanged: false`.
  The retry proving the frame did not change is the payload saying "this is the asset, not the
  warm-up" — the distinction the ticket exists for. `frameWarning` appears only there and says so in
  words, including *"a genuinely black or flat-coloured asset lands here too and is not an error"* and
  *"Do NOT compare this frame against another capture"*. Non-degenerate calls carry no `frameWarning`
  and no `coldFrameRetry`.

  Not exercised: a genuine cold frame that the retry actually repairs (`frameChanged: true` with a
  high `differingFraction`) — forcing one needs a material compile, which is the ~4-minute
  game-thread stall `B-compile-material-blocks-and-mislabels` describes, and this editor is shared.
  The retry mechanism itself is proven to run and to publish both passes. Closing.
