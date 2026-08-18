---
id: B-exposure-pin-black-frame
title: "A pinned capture reports pinned:true / blank:false with no warning while returning an all-black frame — and the wiki's worked example, exposure: 11, is inside the dead zone"
status: OPEN
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
