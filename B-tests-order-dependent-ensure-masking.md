---
id: B-tests-order-dependent-ensure-masking
title: "add_notify/add_notify_state/add_montage_notify instantiate an abstract UAnimNotify/UAnimNotifyState → editor ensure + notify nulled on save"
status: IN-REVIEW
severity: Medium
category: bug
tags: [animation, authoring, notify, ensure, data-loss]
claimedBy: fuzz2
claimedAt: 2026-07-10T19:25:12.3628549+03:00
---

# add_notify / add_notify_state / add_montage_notify instantiate an abstract notify class

The three animation-authoring notify handlers resolve a notify class by name and, on a miss,
fall back to the abstract base class and then `NewObject` it
(`AnimationAuthoringHandler_Sequence.cpp`):

- `animation.authoring.add_notify` — default `notifyClass` "AnimNotify" → synthetic
  "AnimNotify_AnimNotify" → `FindFirstObject` miss → fallback to `UAnimNotify::StaticClass()`.
- `animation.authoring.add_montage_notify` — same fallback.
- `animation.authoring.add_notify_state` — fallback to `UAnimNotifyState::StaticClass()`.

Both base classes are `UCLASS(abstract)` (engine `AnimNotify.h`, `AnimNotifyState.h`).
`NewObject` on an abstract UClass trips the editor-only ensure in
`StaticAllocateObjectErrorTests` (`UObjectGlobals.cpp`: "Class which was marked abstract … It
will be nulled out on save"), and the created notify is **nulled out on save** while the handler
still returns `success:true` — silent data loss on the DEFAULT invocation path, independent of
test order. The non-authoring `animation.add_notify` (`AnimationHandler.cpp`) is the correct
reference: it only `NewObject`s a resolved class, otherwise it creates a valid name-only notify.

Reproduced live (red test): an isolated headless run of
`PinWright.animation.authoring.add_montage_notify.AppliesRequestedTime` captures
`Ensure condition failed: false [UObjectGlobals.cpp]` with a callstack through
`NewObject<UAnimNotify>` at the add_montage_notify site. The engine ensure is one-shot per site
per session, so it only *surfaces* as a test failure under filtered/subset runs (earlier
full-suite tests disarm the site first) — but the null-on-save corruption happens every time.

A second, related editor ensure fires from the same handlers: setting a notify's `TrackIndex`
on a fresh asset that has no notify tracks and calling `RefreshCacheData()` trips
`AnimNotifyTrack … has notify … with track index … that does not exist` (`AnimSequenceBase.cpp`).
The handlers must guarantee the target notify track exists first.

**Fix:** never `NewObject` an abstract class. For add_notify / add_montage_notify, when the
resolved class is null or abstract, create a name-only notify (null `Notify`) — matching the
default "generic notify" intent and the non-authoring reference. For add_notify_state (no valid
name-only state), reject with a concrete-class-required error instead of instantiating the
abstract base. Guarantee the target notify track exists before `RefreshCacheData()`. Root-cause
the trigger; do NOT suppress with `AddExpectedError`.

## Scope / reword note

This ticket was reworded from a symptom-framed original ("8 tests only pass in full-suite
order"). Source verification showed the original bundled four unrelated root causes across the 8
tests: only `add_montage_notify.AppliesRequestedTime` exercises this production abstract-notify
path. The residual transient-`AnimSequence` **test-fixture** ensures (resample subframe +
missing notify track via `NewLoadedTransientAnimSequence`), which drive the order-dependence of
the five dump-parity/list tests, are split to follow-up `F-anim-transient-sequence-fixture-ensures`.
The two non-anim tests in the original set (`asset.dump.WidgetScreenshot.OptInProducesPng`,
`CRIR.RoundTrip.FunctionInterfacePinDefaults`) call no anim code — they were collateral holding
the capture window open when a virgin anim ensure fired, and should clear once the anim ensures
are fixed.

Original symptom set (subset filter
`PinWright.anim+PinWright.AGIR+PinWright.agir+PinWright.CRIR+PinWright.asset.dump+PinWright.ui.activatable`,
`PinWrightTestsSubset.log` vs `PinWrightTestsFull2.log`):
add_montage_notify.AppliesRequestedTime (this ticket) · get_animation_info.AnimSequenceParityFields ·
list_notifies.ReturnsSortedDumpParityShape · list_sync_markers.ReturnsSortedDumpParityShape ·
set_notify_state_property.SetsNestedObjectPath · describe_sequence.ReturnsDumpShape (all → the
follow-up) · asset.dump.WidgetScreenshot.OptInProducesPng · CRIR.RoundTrip.FunctionInterfacePinDefaults
(collateral).

## History
- `#1-subset-run-exposes-masked-ensures` `OPEN` reporter — 8 tests fail under filtered subset runs but pass in full-suite order; engine one-shot ensures (UObjectGlobals.cpp:3447, AnimSequence.cpp:4931, AnimSequenceBase.cpp:679) are disarmed by earlier tests in full runs. Repro: run the subset filter above headless; compare PinWrightTestsSubset.log vs PinWrightTestsFull2.log.
- `#2-reword-to-production-ensure` `IN-REVIEW` developer — REWORD + fix shipped. Source-verified the real defect is a production ensure, not test order-dependence: the 3 authoring notify handlers fell back to instantiating the abstract UAnimNotify/UAnimNotifyState base (UObjectGlobals abstract-class ensure + notify nulled on save; reproduced live). Fix in Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp (add_notify/add_montage_notify → name-only notify when the resolved class is null/abstract; add_notify_state → reject with CLASS_NOT_FOUND since name-only states are invalid) plus new AnimationAuthoringHelpers::EnsureNotifyTrack (grows AnimNotifyTracks so RefreshCacheData never ensures on a missing track). Regression test PinWright.animation.authoring.add_montage_notify.DoesNotInstantiateAbstractNotify (real /Game/Characters/Mannequins/Animations/Manny/MM_Rifle_Reload_Montage fixture) asserts exactly one non-abstract notify is attached: compiles clean, passes post-fix, stash-verified FAILING pre-fix (captured the UObjectGlobals.cpp:3449 ensure via the handler callstack + the assertion). Co-change: converted the latent-ODR anon-namespace FindParamSpec in Tests/Geometry/TestGeometryCreateNameParamAlias.cpp to the shared ParamSpecTestHelpers::FindParamSpec (unblocks adding the Gameplay test under unity). Residual transient-AnimSequence test-fixture ensures (resample + track, driving 5 dump-parity/list tests) split to new OPEN ticket F-anim-transient-sequence-fixture-ensures; the 2 non-anim tests are collateral. Severity stays Medium (matches the DONE short-class/CDO ensure-fix precedents).
