---
id: F-anim-transient-sequence-fixture-ensures
title: "NewLoadedTransientAnimSequence test fixture trips engine ensures (resample subframe + missing notify track) → anim dump-parity tests order-dependent"
status: OPEN
severity: Low
category: bug
tags: [testing, animation, ensure, order-dependence, test-fixture]
---

# Transient-AnimSequence test fixture trips one-shot engine ensures

Split from `B-tests-order-dependent-ensure-masking` (which fixed the production abstract-notify
ensure in the authoring handlers). The remaining order-dependent failures come from the TEST
FIXTURE helper `NewLoadedTransientAnimSequence` (and the manual notify setup) in
`Plugins/PinWright/Source/PinWright/Private/Tests/Gameplay/TestAnimationHandlers.cpp`, which
builds a transient `UAnimSequence` and drives the animation data controller in a way that trips
editor-only, one-shot engine ensures:

- `Ensure condition failed: FMath::IsNearlyZero(ResampledFrameTime.GetSubFrame())`
  (`AnimSequence.cpp`) — the model frame rate does not resample to whole frames. Reachable via
  the fractional/NTSC `FFrameRate(24000, 1001)` used by
  `get_animation_info.AnimSequenceParityFields` (TestAnimationHandlers.cpp).
- `Ensure condition failed: GetOutermost()->bIsCookedForEditor` (`AnimSequenceBase.cpp`) — a
  notify on a track index that does not exist on the fresh transient asset.

Because engine ensures fire once per site per session, these fail only under filtered/subset
runs; in full-suite order an earlier test disarms the site first. Affected tests (pass in full
suite, fail in subset):

- PinWright.animation.authoring.get_animation_info.AnimSequenceParityFields
- PinWright.animation.authoring.list_notifies.ReturnsSortedDumpParityShape
- PinWright.animation.authoring.list_sync_markers.ReturnsSortedDumpParityShape
- PinWright.animation.authoring.set_notify_state_property.SetsNestedObjectPath
- PinWright.animation.describe_sequence.ReturnsDumpShape

The two non-anim tests in the original 8-test set — `asset.dump.WidgetScreenshot.OptInProducesPng`
and `CRIR.RoundTrip.FunctionInterfacePinDefaults` — call no anim code; analysis says they are
collateral that merely held the automation capture window open when a virgin anim ensure fired.
They should stop failing once these anim ensures are fixed. If they persist under subset runs,
file a fresh ticket with the independently-reproduced ensure (do not assume an anim cause).

**Fix direction:** make `NewLoadedTransientAnimSequence` (and the affected tests' notify setup)
cook/resample-safe — pick a whole-frame-compatible frame rate, ensure a notify track exists
before adding notifies, and/or set the transient package's editor-cook flag as appropriate — so
the fixture never trips an engine ensure. Do NOT paper over with `AddExpectedError`; root-cause
the trigger, matching the production fix in `B-tests-order-dependent-ensure-masking`.

**Acceptance:** for each of the 5 tests above, a targeted per-test isolated headless run
(`Automation RunTests <FullTestPath>;Quit`, `-RenderOffScreen -unattended -nopause -nocefaccelpaint`)
shows `Result={Success}` with no captured engine ensure in its BeginEvents..EndEvents window —
i.e. the test passes without relying on an earlier test having disarmed the ensure site.

## History
- `#1-split-from-ensure-masking` `OPEN` developer — split from B-tests-order-dependent-ensure-masking. The production abstract-notify ensure was fixed there; this tracks the residual transient-AnimSequence TEST-FIXTURE ensures (resample subframe AnimSequence.cpp + missing notify track AnimSequenceBase.cpp via NewLoadedTransientAnimSequence) that keep the 5 dump-parity/list tests order-dependent under subset runs. Repro: run the subset filter headless and compare against the full-suite log; or run each affected test in isolation and grep its window for the ensures.
