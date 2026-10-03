---
id: F-anim-transient-sequence-fixture-ensures
title: "set_notify_state_property / list_notifies tests add notifies to a fixture sequence with no AnimNotifyTracks → RefreshCacheData's missing-track ensure, order-dependent under subset runs"
status: OPEN
severity: Low
rice: [1, 1, 1, 1]
priority: 8
category: bug
tags: [testing, animation, ensure, order-dependence, test-fixture]
---

# Notify tests add notifies with no notify track → one-shot engine ensure, order-dependent under subset runs

The tests build their sequence with `PinWrightAnimationTestFixtures::FRegisteredAnimationFixture`
(`Source/PinWright/Private/Tests/Gameplay/TestAnimationFixtures.h:25`), which creates no
`AnimNotifyTracks`. Two tests then append notifies straight to `Sequence->Notifies` with the
default `TrackIndex` 0:

- `PinWright.animation.authoring.set_notify_state_property.SetsNestedObjectPath`
  (`Tests/Gameplay/TestAnimationHandlers.cpp:785-789`). The handler then calls
  `AnimAsset->RefreshCacheData()` (`Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp:1975`),
  and the engine's `UAnimSequenceBase::RefreshCacheData` raises
  `ensureMsgf(GetOutermost()->bIsCookedForEditor, "AnimNotifyTrack: ... track index (0) that does not exist")`
  for each such notify before repairing the track list.
- `PinWright.animation.authoring.list_notifies.ReturnsSortedDumpParityShape`
  (`TestAnimationHandlers.cpp:682-694`) builds the same invalid state. `list_notifies` itself
  does not call `RefreshCacheData` (`AnimationAuthoringHandler_Sequence.cpp:1664-1686`), so
  whether this test trips the ensure is unconfirmed.

An engine ensure fires once per site per session, so the failing test is whichever reaches the
site first: the test fails under a filtered or subset run and passes in full-suite order.

The production handlers already avoid this by calling `AnimationAuthoringHelpers::EnsureNotifyTrack`
before `RefreshCacheData` (`AnimationAuthoringHandler_Sequence.cpp:1749`, `:1838`, `:2641`).

**Fix:** in both tests, create the notify track before adding notifies (call
`AnimationAuthoringHelpers::EnsureNotifyTrack(Sequence, Event)` per event, or have the fixture
add one notify track). Do not suppress with `AddExpectedError`.

**Acceptance:** each of the two tests, run alone headless
(`Automation RunTests <FullTestPath>;Quit`, `-RenderOffScreen -unattended -nopause -nocefaccelpaint`),
shows `Result={Success}` and no `AnimNotifyTrack: ... does not exist` ensure in its
BeginEvents..EndEvents window.

## History
- `#1-split-from-ensure-masking` `OPEN` developer — split from B-tests-order-dependent-ensure-masking. The production abstract-notify ensure was fixed there; this tracks the residual transient-AnimSequence TEST-FIXTURE ensures (resample subframe AnimSequence.cpp + missing notify track AnimSequenceBase.cpp via NewLoadedTransientAnimSequence) that keep the 5 dump-parity/list tests order-dependent under subset runs. Repro: run the subset filter headless and compare against the full-suite log; or run each affected test in isolation and grep its window for the ensures.
- `#2-rephrased` `OPEN` developer — Old text blamed NewLoadedTransientAnimSequence for two ensures across 5 tests and listed two collateral non-anim tests. At 7230b41d all 5 tests use FRegisteredAnimationFixture; the resample-subframe ensure is avoided by its 800-frame count for 24000/1001 (TestAnimationFixtures.h:89-96), and get_animation_info parity, list_sync_markers and describe_sequence add no notifies. Narrowed to the remaining missing-notify-track ensure: set_notify_state_property test adds a notify with no AnimNotifyTracks and its handler calls RefreshCacheData (AnimationAuthoringHandler_Sequence.cpp:1975); list_notifies builds the same state but is unconfirmed. Collateral-test paragraph dropped. Severity stays Low. rice 1 1 0.5 1 -> 1 1 1 1: the set_notify_state_property path to the ensure is now verified in source.
