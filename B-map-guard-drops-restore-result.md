---
id: B-map-guard-drops-restore-result
title: "FScopedEditorWorldMapGuard's destructor restores the editor map through level.load and discards the result, so a refused restore is invisible and strands the editor on a torn-down world for every later test in the run"
status: OPEN
severity: Medium
category: bug
tags: [tests, test-harness, scoped-guard, teardown, level-load, misattributed-failure, silent-failure, InvokeHandler, safe-point]
encounters: 1
lastSeen: 2026-09-03T10:20:21Z
---

# A teardown that calls a verb, ignores whether it worked, and blames the next two tests

`FScopedEditorWorldMapGuard` (`Source/PinWright/Private/Tests/TestWorldUtils.h:150-189`) is the
harness's map-swap safety net: 12 call sites in 4 test files construct it so that a test which
swaps the active editor world puts the original map back on every exit path. Its destructor
(`:161-178`) does the restore by invoking the production verb:

```cpp
        UWorld* const FinalWorld = GEditor->GetEditorWorldContext().World();
        if (FinalWorld != OriginalWorld && FPackageName::DoesPackageExist(OriginalMapPath))
        {
            TSharedPtr<FJsonObject> RestorePayload = MakeShared<FJsonObject>();
            RestorePayload->SetStringField(TEXT("levelPath"), OriginalMapPath);
            InvokeHandler(TEXT("level.load"), RestorePayload);   // <-- :176, return value discarded
        }
```

**Nothing observes the outcome.** `InvokeHandler`'s return value is discarded, and it would not
help anyway: `Tests/TestUtils.h:206-219` returns "the method was found in the registration list",
never "the call succeeded". The verb's own response is discarded twice over, because
`InvokeHandler` builds the context through `FHandlerContext::MakeTestContext`, whose subsystem
argument defaults to `nullptr` (`Handlers/HandlerContext.h:171-175`) and which sets no
`ResponseCapture` (`Handlers/HandlerContext.cpp:647-659`). A `SendError` from `level.load`
therefore falls through both branches to the drop path at `HandlerContext.cpp:458`:

```
HandlerContext::SendError called with no Subsystem and no ResponseCapture - response dropped
```

That line names neither `level.load`, nor the error code, nor the map. It is one word different
from the success-path line at `HandlerContext.cpp:404`, which every *working* restore also emits.
So the entire signal that a restore failed is a `LogTemp` warning distinguishable from the healthy
case only by the substring `SendError`.

## What it cost, in a shipped run

Recorded in `B-level-load-dirty-world-fatal` ("Fix correction (post-suite)"). That ticket's first
cut of the `MapSwapDirtyWorldGuard` precondition was over-broad and refused a restore this guard
was performing. Consequence, from `Saved/Logs/pw_wave_suite.log` at `07.22.05`-`07.22.06`:

- `level.load` refused with `DIRTY_WORLD_BLOCKS_MAP_SWAP`; the response was dropped
  (`HandlerContext::SendError called with no Subsystem and no ResponseCapture` at `07.22.05:118`,
  with **no** `Cmd: MAP LOAD FILE=` following it).
- The test whose teardown this was **passed**.
- The editor stayed on the torn-down probe world left behind by `level.structure.create_level`.
- Two *unrelated* tests several positions later failed against it:
  `PinWright.level.structure.create_level_instance.SetsWorldAsset` and
  `...create_packed_level_actor.PacksSourceLevel`.

The reds pointed at two level-instance verbs that had nothing wrong with them; the defect was one
precondition and one silent teardown. Diagnosis had to be done backwards from the log, and the
parent ticket ends by saying so: *"That cost the whole diagnosis here. Worth its own ticket."*

## Why it recurs

`level.load` is not a stable target and is not meant to be. Since this guard was written it has
grown two preconditions that can refuse or postpone the call, both of which exist to stop the
editor process from dying:

- `PinWrightSafePoint::RunAtSafePoint` (`Handlers/Level/LevelHandler.cpp:291`) - tick-reentrancy,
  filed as `B-open-asset-world-map-load-crash`.
- `PinWrightMapSwapGuard::WouldMapLoadFatal` -> `DIRTY_WORLD_BLOCKS_MAP_SWAP`
  (`LevelHandler.cpp:329`) - the unconditional `appError` at `EditorServer.cpp:2544`, filed as
  `B-level-load-dirty-world-fatal`.

Every future precondition inherits the same blind spot. The guard is one line of coupling to a
verb whose refusal surface is expected to keep growing.

## A second, latent failure mode in the same call

`RunAtSafePoint` returns `true` on both branches (`Dispatch/SafePoint.h:452-472`): inline, or after
parking the work on the next core-ticker pass. On the deferred branch the destructor returns with
the map **not yet restored**, and nothing in the destructor pumps the core ticker or waits. The
guard's contract ("on destruction, the map is back") is not merely unverified there, it is not
satisfied.

Stated as latent rather than active on purpose: the one test that deliberately forces the deferred
branch, `Tests/World/TestLevelLoadSafePoint.cpp:149`, scopes its `FScopedForcedUnsafeMapSwap`
inside the guard's lifetime and pumps before returning, so the latch is released before the
destructor runs. Any future test that forces unsafe (or a real `IsSafeNow()` false at teardown)
turns it active, with the same silent shape.

## Census: how far the pattern spreads

A brace-balanced scan of every `~F...()` destructor in every `Private/Tests/` tree (main module and
all five integration sub-modules) found **exactly one** destructor whose body calls `InvokeHandler`
or `InvokeHandlerWithCapture` - this one. Every other teardown goes through engine APIs rather than
a verb: `FScopedEditorWorldActorGuard` (`TestWorldUtils.h:78-126`, `EditorDestroyActor` +
`SetDirtyFlag`), `FScopedLevelLock` (`:199-230`, `FLevelUtils::ToggleLevelLock`),
`PwTestAssetTeardown::DiscardCreatedAssetByObjectPath`, `CleanupTestAsset` (`TestUtils.h:530`),
`FScopedRegisteredSequence` (`Tests/Media/TestSequencerHandlers.cpp:79`), and the console-variable
guards. So this is a one-class fix, not a sweep - but it is the only teardown in the tree whose
correctness depends on a verb's preconditions, which is exactly why it should not stay unchecked.

Also note what does *not* apply: `PinWrightTestSkip` (`Tests/TestSkipReporting.h`) is the channel
for **stepped-over assertions**, not for a teardown that failed. A failed restore is a real error
and must red something; routing it through the skip marker would report "this run measured less"
when the truth is "the rest of this run ran against the wrong world."

## Affected call sites (12)

```
Source/PinWright/Private/Tests/EditorOps/TestOpenAssetWorldNoCrash.cpp:129
Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp:107, 417, 460, 538, 1042, 1387, 1510
Source/PinWright/Private/Tests/World/TestLevelLoadSafePoint.cpp:149
Source/PinWright/Private/Tests/World/TestLevelSavePathTargeting.cpp:280, 393, 510
```

All 12 are bare default constructions (`FScopedEditorWorldMapGuard MapGuard;`), so any signature
change is mechanical but touches all of them.

## Fix

**Recommended: surface the failure and attribute it - do NOT bypass the verb.** Three parts, all
inside `TestWorldUtils.h` plus the 12 mechanical call-site edits:

1. **Call `InvokeHandlerWithCapture`, not `InvokeHandler`.** Today the error code and message do
   not exist anywhere - the capture is what makes them exist at all.
2. **Take `FAutomationTestBase&` in the constructor and `AddError` on failure**, naming
   `OriginalMapPath`, `Capture.ErrorCode` and `Capture.Message`. The destructor runs inside
   `RunTest` at all 12 sites (the guard is a function local), so the error lands on the test that
   caused the pollution rather than on whatever runs next. Pair it with one
   `UE_LOG(LogPinWright, Error, ...)` so the log carries it even where the automation event does
   not survive.
3. **Assert the restore actually happened**, not just that the handler was found: after the call,
   compare `GEditor->GetEditorWorldContext().World()->GetOutermost()->GetName()` against
   `OriginalMapPath`. This is the only part that also catches the deferred-safe-point mode above,
   because that branch returns success-shaped nothing.

**Rejected: restoring through `FEditorFileUtils::LoadMap` directly to dodge the preconditions.**
It reads as the clean fix - the guard is test infrastructure, not a user call - and it is the wrong
trade. The two preconditions it would bypass are not policy, they are the guards against a
`check()` failure and an unconditional `UE_LOG(Fatal)` that each kill the whole editor process and
with it the entire suite run. A raw `LoadMap` from a teardown reintroduces both in the place where
they are hardest to attribute. It also drops the path resolution the guard's own header comment
(`TestWorldUtils.h:146-148`) says it went through the verb to get. Converting a misattributed test
failure into a suite-killing crash is a strictly worse outcome.

**Optional 4th part, if the refusals turn out to be self-inflicted.** When the block is the
harness's own doing - the guard's test dirtied the probe world it is now leaving - clear that one
condition and retry once *through `level.load`*, mirroring what the sibling
`FScopedEditorWorldActorGuard` already does with `SetDirtyFlag` at `TestWorldUtils.h:121-125`.
Never a raw `LoadMap`.

**Trade-off, stated plainly.** After part 2 a legitimately-refused restore reds the test that ran
it. That is the intended behaviour, not a regression: a run whose editor is on the wrong world is
not a green run, and one red at the source beats two reds three tests later. It does mean a future
precondition tightened for good reasons will surface as a test failure - which is the whole point,
and is how it would have been caught in `pw_wave_suite.log` in one step.

**Not a fix, and worth saying:** raising the drop-path message at `HandlerContext.cpp:458` to
`Error`, or making it name the method, would help every caller of the no-capture `InvokeHandler`
but would not fix this - a warning nothing asserts on is still a warning nothing asserts on. Note
also that whether that line is even visible is host-config dependent: `AutomationTest.cpp:181`
defaults `bElevateLogWarningsToErrors` to `false` and reads an override from `GEngineIni`
(`:2055`), while `UAutomationControllerSettings`'s CDO sets it `true`
(`AutomationControllerSettings.cpp:14`); this host pins it `false`
(`unreal-fpv-dev/Config/DefaultEngine.ini:410`). On a host that leaves it `true` the same
uninformative warning fails the enclosing test instead - loud, but naming neither the guard nor
`level.load`. Neither direction is a diagnosis.

severity rationale: impact = silent false-success in harness teardown (the guard reports nothing
while having failed, and the run continues against the wrong editor world, misattributing failures
to unrelated tests) x reach = only fires when a `level.load` precondition refuses or defers the
restore, a rare edge path today though one guaranteed to recur as preconditions accumulate ->
bump down from High -> **Medium**. Matches the band of `B-tests-spawn-live-world-no-guard`, the
same "test-infra guard does not do the job it exists for" class. Deliberately not High: no product
defect is hidden by it and no coverage is lost, unlike `B-test-skips-assertions-silently` and
`B-test-ids-swallowed-by-dot-prefix`.

## History
- `#1-found-during-dirty-world-guard-fix` `OPEN` reporter - Split out of `B-level-load-dirty-world-fatal`, whose "Fix correction (post-suite)" section ends by saying this deserves its own ticket. Evidence read from source, not inferred: the discard is `Tests/TestWorldUtils.h:176` (`InvokeHandler(TEXT("level.load"), RestorePayload);`, return value unused); `InvokeHandler` returns handler-found, not success (`Tests/TestUtils.h:206-219`); the context it builds has a `nullptr` subsystem by default (`Handlers/HandlerContext.h:171-175`) and no capture (`HandlerContext.cpp:647-659`), so `SendError` reaches the drop path at `HandlerContext.cpp:458` and emits a message naming neither the verb, the code, nor the map - one word from the identical success-path line at `:404`. Shipped consequence recorded in `pw_wave_suite.log` at `07.22.05`-`07.22.06`: the refused restore, no `Cmd: MAP LOAD FILE=` after it, the teardown's own test green, and `level.structure.create_level_instance.SetsWorldAsset` + `create_packed_level_actor.PacksSourceLevel` red several positions later. Second failure mode found while reading: `RunAtSafePoint` returns true on both branches (`Dispatch/SafePoint.h:452-472`), so a deferred restore leaves the destructor returning with the map unrestored and nothing pumping the ticker - latent today only because `Tests/World/TestLevelLoadSafePoint.cpp:149` scopes its forced-unsafe latch inside the guard. Census: a brace-balanced scan of every destructor in every `Private/Tests/` tree found this to be the ONLY destructor in the plugin that invokes a verb, so the fix is one class rather than a sweep. Recommending options 1+3 of the three considered and explicitly rejecting the "bypass with `FEditorFileUtils::LoadMap`" option, which would trade a misattributed failure for the two process-killing fatals those preconditions exist to prevent (`B-open-asset-world-map-load-crash`, `B-level-load-dirty-world-fatal`).
