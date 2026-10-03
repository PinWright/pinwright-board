---
id: B-headless-skips-cpu-only-assertions
title: "The null-rhi skip drops whole tests whose assertions are partly or wholly CPU-only, so headless suite runs lose registration, schema and parameter-validation coverage"
status: OPEN
severity: Low
category: bug
tags: [tests, nullrhi, headless, test-skips, coverage, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
rice: [1, 1, 1, 2]
priority: 4
---

# Null-rhi skips cost CPU-only coverage in headless runs

`B-headless-nullrhi-renderer-dependence` put `PinWrightTestSkip::SkipIfRenderingUnavailable` as the
first statement of ~196 tests (list in that ticket's body and `#3`/`#4`). For several of them only
part of the test needs a renderer, so a headless run now skips assertions it could have measured.
Verified examples:

- `infra.dispatcher.NestedInputClosure.ConfirmedCasesRefuseBeforeLoad`
  (`Tests/Infra/TestNestedParamKeyGate.cpp:616-621`): six method families (animation, material,
  render, blueprint, networking, eqs) checked against the dispatcher's nested-key validator. Only the
  `render.capture_annotated` case that expects to reach the handler depends on the renderer; the
  other five families are pure dispatcher validation.
- `effect.step_and_capture.ContractAndStageOrder` (`Tests/Niagara/TestEffectStepAndCapture.cpp:185`):
  registration, the `frames` parameter type and the tick-unsafe flag (`:188-200`) are registry reads.
- `render.detect_z_fighting.RejectsPowerOfTwoNearPlaneRatio`
  (`Tests/Render/TestZFightingDetect.cpp:228`): the closing
  `PinWrightZFighting::IsPowerOfTwoRatio(DefaultNearPlaneRatio)` assertion is pure CPU; the ratio
  rejections would be too if validation ran before the handler's renderer guard.
- `ui.screenshot.ValidParamsNoCrash` (`Tests/Media/TestUIHandlers.cpp:109-110`): the whole test is
  "handler found", which `InvokeHandler` returns true for even when the handler refuses with
  `RENDERING_UNAVAILABLE`; the skip removes a test that needs no renderer at all.
- The parameter-validation tests in the list (e.g. `landscape.sculpt.MissingRequiredParam`,
  `landscape.sculpt.RejectsUnknownToolMode`, `render.capture_open_level.InvalidDimensions` / `InvalidProjectionMode`,
  `render.capture_annotated.InvalidGrid`, `render.capture_ortho_tiles.AxesRequired`,
  `camera.animation_shots.RequiresSequencePath`, `mrq.run_jobs.RejectsEmptySelection`) only need a
  renderer because each handler calls `RequireRenderer` as its first statement, before parsing
  parameters (e.g. `Handlers/Environment/LandscapeHandler.cpp:966`, `:1851`).

Offscreen and visible runs still execute every one of these, so no coverage is lost there.

**Impact:** a headless suite verdict (COMPLETED_WITH_SKIPS) measures less than it could; a
regression in these CPU-only checks is invisible to headless-only CI. Test-only; no user impact.
**Fix:** split mixed tests into a CPU part (no skip) and a renderer part (null-rhi skip); delete the
skip from `ui.screenshot.ValidParamsNoCrash`. For the validation tests, either move `RequireRenderer`
below pure parameter parsing in those handlers (a headless caller with bad params then gets the
parameter error first, which is also more useful) or keep the guard first and accept the loss.

## History
- `#1-mixed-tests-skip-whole` `OPEN` reporter — Found in today's verification of the headless work; each cited test read at the given lines. Follow-up to `B-headless-nullrhi-renderer-dependence` (IN-REVIEW), filed separately so its verification is not blocked on the split.
