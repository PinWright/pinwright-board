---
id: B-exposure-pin-black-frame
title: "A pinned capture reports pinned:true / blank:false with no warning while returning an all-black frame — and the wiki's worked example, exposure: 11, is inside the dead zone"
status: IN-REVIEW
severity: High
category: bug
tags: [render, capture_asset_preview, exposure, exposure-pin, blank, silent-success, docs, determinism]
encounters: 1
lastSeen: 2026-08-18T00:00:00Z
---

# A pinned capture reports success while returning an all-black frame

`render.capture_asset_preview` with an `exposure` pin returns `exposure.pinned: true`,
`imageStats.blank: false` and **no warning of any kind** for a frame that is crushed to black
and carries no usable information. The caller cannot distinguish a good capture from a useless
one, and any downstream comparison then operates on black — the "reports success while doing
nothing" shape.

Measured on `render.capture_asset_preview` against the `FAdvancedPreviewScene` fixture, sweeping
`ev100` and reading mean luminance off the returned frame:

| EV100 | meanLuminance |
|---|---|
| −1 | 0.3644 |
| 0 | 0.2466 |
| 5 | 0.0162 |
| 10 | 0.0068 |
| 16 | 0.0068 |

Usable range on this fixture is roughly EV100 −1..0. By EV100 5 the frame is at 6.6% of the
EV100 0 luminance; from EV100 10 upward it is pinned at the floor (0.0068) and stops responding
to the parameter at all — every value from 10 to 16 returns the same black.

**The wiki actively recommends a value in that dead zone.** `Docs/wiki-src/render.md:47` and
`Docs/wiki-src/camera.md:37` both give `exposure: 11` as the worked example. A caller who copies
the documented example gets mean luminance ~0.007, `pinned: true`, and no signal that anything
is wrong. (Another agent is reported to be fixing these two lines concurrently; recorded here as
the state observed at filing.)

## Why nothing catches it

Three independent gaps, each individually defensible:

1. **`pinned` is a predicate over viewport state, not over the frame.**
   `ExposurePinGovernsFrame` (`PreviewViewportCaptureUtils.cpp:250-276`) tests exactly three
   things — the `PostProcessing` show flag, the `Lighting` show flag, and lit-ness of the view
   mode — and answers "would the renderer apply this override". That is the right question for
   the field's stated contract, and its result is technically accurate. It is simply uncorrelated
   with "the resulting frame is usable", so `pinned: true` reads as reassurance it was never
   designed to give. The `pinWarning` path (`:1440`) fires only for the inverse case (pin
   requested, renderer ignored it); nothing fires for a pin that took and produced black.

2. **The blank classifier is a conjunction, and the crushed frame fails one half.**
   `Stats.bBlank = MeanLuminance <= 0.01 && LuminanceVariance <= 0.0001`
   (`PreviewViewportCaptureUtils.cpp:353`). At EV100 11 the mean (0.0068) is comfortably under
   the first gate, but a crushed frame keeps enough residual variation to clear the second, so
   `blank: false`. The classifier is deliberately scoped to "black frame, not boring image"
   (comment at `:350-352`) — it was built for a *dead readback*, not for *correct pixels rendered
   with an unusable exposure*, and those are different failures.

3. **`capture_asset_preview` never applies the blank gate anyway.**
   `bRejectBlankCapture` is set at exactly one site, `RenderHandler.cpp:474`, inside the
   `render.capture_open_level` registration (`:386-515`). `render.capture_asset_preview`
   (registered `:212`) computes and reports `imageStats` but never rejects or retries on them.
   So even a frame that *did* trip `bBlank` would come back as a clean success from this verb.

Net: the pin succeeded, the renderer honoured it, the pixels are "correct" — and the capture is
worthless. Every reported field is true and the aggregate is misleading.

## Second finding: a pinned capture is not reproducible, so equality is the wrong comparison

Recorded here because it lands on the same contract — what a caller may conclude from
`pinned: true`. **Pinning exposure does not make two captures byte-identical.** Two back-to-back
captures at the same EV100, same camera, same asset:

- 11676 of 16384 px differ
- mean absolute difference **0.88** (8-bit levels), max 34
- best-fit scalar gain **1.0000** — so this is temporal jitter, not residual exposure drift

Independently consistent with a separately measured **1.30% viewport self-noise floor**. For
scale, the signal from a real one-stop-plus exposure change measures ~**89.6** mean absolute
difference: noise and signal are ~100x apart, so the noise is harmless *if* the comparison uses
a tolerance and fatal if it uses equality. Nothing in the docs says so today, and the phrasing
"a burst is internally consistent by construction" (`render.md:47`) invites the equality reading.

## Fix

- **Measure the frame the caller is actually holding, and warn when it is degenerate.** Same
  spirit as the existing `blank` / `IsBlankReadback` check, but for "crushed to black or blown
  out" rather than "no readback": the stats needed (`MeanLuminance`, `MinLuminance`,
  `MaxLuminance`, `LuminanceVariance`) are already computed in
  `CalculateCaptureImageStats` (`:321-355`) and already in the response. A warning string in the
  `exposure` block (alongside `pinWarning` / `restoreWarning`) is the cheapest correct shape — it
  does not change any verb's success/failure contract, and a deliberately dark subject stays
  capturable. Prefer a warning over a hard `ERR_BLANK_CAPTURE` here: unlike a dead readback, a
  dark frame can be intentional.
- **Do not widen `bBlank`'s conjunction to cover this.** Its two-term form is load-bearing for
  the dead-readback case; a crushed-but-textured frame is a different classification and should
  get its own field rather than being folded into `blank`.
- **Fix the two wiki examples.** `Docs/wiki-src/render.md:47` and `Docs/wiki-src/camera.md:37`
  must not use `exposure: 11`. Pick a value inside the usable band and, better, say in the same
  paragraph that the usable `ev100` band is scene-dependent and that the returned
  `imageStats.meanLuminance` is how you check you picked one.
- **Document the tolerance.** State in `render.md` that pinned captures are not byte-identical
  (~0.88 mean abs diff, ~1.3% self-noise), so a comparison must use a tolerance well below the
  ~89.6 signal level of a real one-stop change, never equality.

## History
- `#1-initial-measurement` `OPEN` reporter — Measured `render.capture_asset_preview` against the `FAdvancedPreviewScene` fixture across an EV100 sweep: meanLuminance 0.3644 (EV100 −1), 0.2466 (0), 0.0162 (5), 0.0068 (10), 0.0068 (16) — pinned at the floor and unresponsive to the parameter from EV100 10 up, while the response reported `pinned: true`, `blank: false` and no warning. Traced why nothing catches it: `ExposurePinGovernsFrame` (`PreviewViewportCaptureUtils.cpp:250-276`) is a three-term predicate over viewport-client state (PostProcessing show flag, Lighting show flag, lit view mode) that is accurate about the renderer and uncorrelated with frame usability; `bBlank` (`:353`) requires mean ≤ 0.01 **and** variance ≤ 0.0001, and a crushed frame clears the variance term; and `bRejectBlankCapture` is set only at `RenderHandler.cpp:474` inside `render.capture_open_level` (`:386-515`), so `render.capture_asset_preview` (`:212`) never applies the gate at all. Compounding it, both wiki worked examples — `Docs/wiki-src/render.md:47` and `Docs/wiki-src/camera.md:37` — recommend `exposure: 11`, which is inside the dead zone (a concurrent agent was reported to be editing those lines at filing time). Also measured and recorded in the same ticket: pinning does **not** make captures byte-identical — two back-to-back same-EV100 captures differ in 11676/16384 px, mean abs diff 0.88, max 34, best-fit gain 1.0000 (temporal jitter, not exposure drift), consistent with a separately measured 1.30% viewport self-noise floor, against a ~89.6 mean-abs-diff signal for a real one-stop-plus change — so pinned-capture comparisons need a tolerance, not equality, and the docs do not say so. Deduped against the board: `B-open-level-blank-success` (IN-REVIEW) covers uniform-black classification on `render.capture_open_level` and is the origin of the `bBlank` gate, but does not cover the crushed-not-uniform case, the `pinned` misreport, or `capture_asset_preview`; `B-capture-asset-preview-renders-empty` (IN-REVIEW) is the non-realtime white-frame bug, a different cause and a different frame; `F-ortho-tile-reference-compare` (OPEN) needs a shared exposure pin for tile bursts and predates the `exposure` parameter existing. Severity High per the rubric's silent-false-success clause on a normal visual-review path; no reach bump applied, and the rubric reserves Critical for crashes and data loss.
- `#2-additional-current-core` `OPEN` reporter — Additional evidence: **Adversarial review A — core defect remains, with stale subclaims.** Actuality: **PARTIAL**. Framing: the canonical `X:\src\unreal\unreal-fpv\Plugins\PinWright` HEAD still accepts a fixed EV100 that crushes the preview fixture while `pinned` only reports renderer-state governance; `CalculateCaptureImageStats` deliberately requires both low mean and low variance for `blank`, and the current exposure warning is emitted only when the renderer ignores the pin (`PreviewViewportCaptureUtils.cpp:321`, `:350-353`, `:1004-1009`, `:1440-1454`). The live calibration/test records EV100 10/16 at mean 0.0068 and explicitly skips pixel assertions for dark output rather than surfacing a product warning (`Source\PinWright\Private\Tests\Render\TestCaptureExposurePin.cpp:668-718`, `:822-883`); runtime reproduction was **NOT VERIFIED** in this review. The `exposure: 11` examples remain (`Docs\wiki-src\render.md:47`, `Docs\wiki-src\camera.md:37`), but the tolerance/equality subclaim is stale: `render.md:55` now documents the 0.88 noise floor and tolerance. Proposed fix: **INCOMPLETE**, the shared warning location is systemic but needs an explicit, tested frame-degeneracy signal/threshold and must preserve intentional dark captures without widening `blank`; `RenderHandler.cpp:472` still makes blank rejection open-level-only. Evidence: `C:\UE_5.8\Engine\Source\Runtime\Renderer\Private\PostProcess\PostProcessEyeAdaptation.cpp:493-510`, `:635-648` confirms fixed exposure can be correctly applied while debug/show-flag branches ignore it. Recommendation: **REFRAME**; retain the open ticket for degenerate pinned-frame reporting and the stale `11` examples, remove the already-fixed tolerance claim, then add a shared static regression plus a guarded interactive preview check.
- `#3-additional-adversarial-scope` `OPEN` reporter — Additional evidence: **Adversarial review B — the source confirms a missing advisory, but A overstates it as an unobservable false-success.** Actuality: **PARTIAL**. Framing: the canonical `X:\src\unreal\unreal-fpv\Plugins\PinWright` HEAD still emits `pinWarning` only for a pin the renderer ignored (`Source\PinWright\Private\Handlers\Render\PreviewViewportCaptureUtils.cpp:1414-1455`), while `bBlank` intentionally requires both low mean and low variance (`:321-353`); however the success response already publishes mean, variance, min and max luminance (`Source\PinWright\Private\Handlers\Render\RenderHandler.cpp:138-145`), so a caller can see that EV100 produced a dark but valid fixed-exposure frame. UE 5.8 applies the requested EV100 in the fixed branch (`C:\UE_5.8\Engine\Source\Runtime\Renderer\Private\PostProcess\PostProcessEyeAdaptation.cpp:493-517`, `:635-648`), making “all-black” scene-dependent rather than intrinsically erroneous. The older `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright` mirror has the same classifier and open-level-only rejection (`Source\PinWright\Private\Handlers\Render\PreviewViewportCaptureUtils.cpp:187-219`, `RenderHandler.cpp:211-212,472-476`) but no exposure-pin implementation/test. The `exposure: 11` examples remain at `Docs\wiki-src\render.md:47,321` and `Docs\wiki-src\camera.md:37`; the equality/noise claim is already corrected at `render.md:55`. High severity is too strong: this is an advisory/docs gap with an explicit caller-controlled EV100 and readable stats, so Medium is the upper bound absent a hard-blocker repro. Proposed fix: **INCOMPLETE**, a threshold warning in `MakeExposureInfoObject` would cover viewport verbs but is scene-dependent and misses the separate scene-capture exposure block (`Source\PinWright\Private\Handlers\Render\OrthoTileCaptureHandler.cpp:658-681`) and its exact-zero-only warnings (`:770-792`); it risks false warnings for intentional dark subjects. Evidence: current calibration records the dark branch as NOT MEASURED rather than a product failure (`Source\PinWright\Private\Tests\Render\TestCaptureExposurePin.cpp:672-718,827-883`). Runtime: **NOT VERIFIED** (no Unreal/editor launch permitted). Recommendation: **REFRAME**; downgrade severity, correct every EV100=11 example and renderer wording, remove the stale tolerance subclaim, and first define/test an advisory low/high dynamic-range contract across both capture paths before implementation.
- `#4-frame-usability-reported-and-gated` `IN-REVIEW` developer — Fixed, in the shape this ticket's own Fix section asked for, plus one defect the fix itself introduced and a second commit removed. This entry exists partly because the fix landed as `08bfb955` and nobody moved the ticket: it sat `OPEN` with no history while the code was already in. **The frame is now measured, and a degenerate one says so.** `08bfb955` added a tone-range classifier beside the existing blank criterion rather than widening it — the ticket was explicit that `bBlank`'s two-term conjunction is load-bearing for the dead-readback case, and that a crushed-but-textured frame is a different classification deserving its own field. It counts how many of 256 8-bit luminance levels the pixels populate and calls fewer than eight a collapse, splitting it into `crushed` (dark half) and `blownOut` (bright half) so the label names which way `ev100` has to move. Published top-level beside `blank` on the three single-frame verbs, with `toneLevelsUsed` and `toneLevelMinPixels` in `imageStats` so no caller has to inherit the threshold, and a `rangeWarning` string quoting the measured count, the mean, and the direction — the direction because `ev100` runs backwards from brightness and shipping only the gain once sent a caller several stops the wrong way. **And the field this ticket's gap analysis really named.** Point 1 of "Why nothing catches it" is that `pinned` is a predicate over viewport state, accurate about the renderer and uncorrelated with frame usability. That is now stated in the response rather than left for a caller to know: `viewport.exposure.pinnedFrameUsable` is the same question asked about THE FRAME, and `pinRangeWarning` fires on exactly the case nothing covered — pin requested, pin applied, renderer honoured it, frame unreadable — since `pinWarning` only ever covered the inverse. `pinned`'s meaning is deliberately unchanged: callers branch on it today and quietly narrowing it would have broken them silently. Both fields shipped undocumented, with zero hits in `Docs/wiki-src/`; `84e21777` documents them with their presence rules and a read-in-this-order ladder, on `render.capture-exposure`. **The fix's own defect.** The same commit that added the classifier added the scoped `viewMode` capture parameter, so wireframe, unlit and debug-visualisation frames began routing straight into it — and those resolve two or three tone levels *because that is what the mode was asked to draw*. A correct two-tone wireframe came back `crushed: true` at full confidence, which is this ticket's shape inverted: not a bad frame reported good, but a good frame reported bad, on the verb an agent uses to decide whether to re-shoot. `a5169b02` gates the verdict on a lit view mode, reusing `IsLitViewMode` through the `bLitViewMode` the capture already measures and reports as `viewport.lit` rather than writing a second predicate that would drift; that is the same term that makes `pinnedFrameUsable` structurally safe, since `ExposurePinGovernsFrame` refuses to pin a non-lit frame at all. A non-lit capture reports `toneRangeApplicable: false` plus a `toneRangeNotApplicable` sentence naming the mode and the remedy — not-applicable as a real answer, following `geometry.audit_static_meshes`' per-check tallies, because omitting `crushed` would read as "measured and fine". The measurement is not withheld: `toneLevelsUsed` is published on a wireframe frame exactly as on a lit one. Regression coverage in `PinWright.render.tone_range.*` (`Tests/Render/TestCaptureToneRangeCriterion.cpp`), which asserts each fixture really does trip the classifier before asserting the gate suppresses it, so the assertions cannot be vacuous. **The docs half.** `367dd9f3` replaces `exposure: 11` in `Docs/wiki-src/render.md` and `Docs/wiki-src/camera.md` with `-1`, inside the measured usable band, and states the thing that generalises: the band is scene-dependent, so pick one by reading `imageStats.meanLuminance` back rather than reusing a number from elsewhere. Both entries `#2` and `#3` recorded those examples as still present; they were. The `render.capture_ortho_tiles` example at `render.md:505` keeps `11` deliberately — it shoots an outdoor level from above, a different scene with a different band, and no measurement here contradicts it. **On the severity dispute.** Entry `#3` argued High is too strong because the response already published mean/variance/min/max, so a caller could see the frame was dark. That reading understates it on one point and is now moot on the other: the caller could see the numbers but nothing said what they meant, and the documentation was actively steering callers to the value that produced them — a worked example inside the dead zone is not a caller-controlled choice in any useful sense. High stands as filed. **Not addressed, and the reason this is IN-REVIEW rather than complete:** the tolerance/equality subclaim both reviews called stale is indeed already documented, and `bRejectBlankCapture` is still set only on `render.capture_open_level` — deliberately, since the ticket itself argues a dark frame can be intentional and warrants a warning rather than a rejection, but a tester should confirm that reading. Verification: capture the preview fixture at `ev100 11` and check the response carries `crushed: true`, a `rangeWarning`, and `viewport.exposure.pinnedFrameUsable: false` with a `pinRangeWarning`; then capture the same asset with `viewMode: "Wireframe"` and check it carries `toneRangeApplicable: false` and NO `crushed`.
