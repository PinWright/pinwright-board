---
id: B-suite-host-gc-crash-in-combined-group-run
title: "A multi-group `Automation RunTests` run kills its own host in garbage collection at ~305 tests — 304 succeeded, 0 failed, no terminal marker, and every group passes clean when run on its own"
status: OPEN
severity: High
category: bug
tags: [tests, suite, automation, garbage-collection, crash, host-stability, cross-test-contamination, false-green, scoped-runs]
encounters: 1
lastSeen: 2026-08-28T08:45:00+05:00
---

# The suite host dies mid-run when groups are combined, and the log looks like a pass right up to the crash

A single `Automation RunTests` invocation with fifteen `+`-joined group filters crashed its own
editor host after 305 tests started and 304 completed successfully. **Zero tests failed.** The run
produced no `N tests performed` drain marker and no `TEST COMPLETE`, so the only thing separating
this from a green run is the absence of a marker — `check_suite_log.py` is what catches it, and the
process exit code (3) is not, because a caller reading exit codes sees "non-zero" for a run whose
last recorded test result was a success.

Each constituent group passes cleanly on its own. This is a **host-stability / cross-test
contamination** defect, not a defect in any one test.

## Measurement

UE 5.8, this checkout, PinWright at `b79ba53e` freshly rebuilt (`UnrealEditor-PinWright.dll`
40,644,096 bytes, 2026-08-28 08:11:48, canonical link).

**Run A — fifteen groups combined.** `DID_NOT_COMPLETE`.

```
filter: PinWright.Model+PinWright.Sequencer+PinWright.actor+PinWright.niagara+PinWright.render
        +PinWright.widget+PinWright.property+PinWright.container+PinWright.texture
        +PinWright.material+PinWright.lighting+PinWright.bpir+PinWright.blueprint
        +PinWright.infra+PinWright.system

found=2335 started=305 succeeded=304 failed=0 skipped=0 performed=None
drainMarker=False testExit=False testComplete=False
```

Crash at `2026.08.28-03.28.02:155` UTC, on a **background worker thread**, during garbage
collection:

```
Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x000000000000000c
  UE::GC::TReferenceBatcher<FMutableReference,FResolvedMutableReference,
      TReachabilityProcessor<5>>::DrainValidatedFull()   GarbageCollection.cpp:1844
  ... ::DrainUnvalidatedFull()                           GarbageCollection.cpp:1843
  UE::GC::TFastReferenceCollector<...>::ProcessObjectArray()  FastReferenceCollector.h:938
  UE::GC::FRealtimeGC::CollectReferencesForGC<...>       GarbageCollection.cpp:4195
  ...
Crash in runnable thread Background Worker #13
FPlatformMisc::RequestExitWithStatus(1, 3, ...)
```

Reading `0x0c` off a null base inside the reachability batcher is the signature of a **stale
`UObject*` reachable from a GC-visible container** — something registered a reference and was
destroyed, or a raw pointer outlived its object. GC ran on a worker thread, so the frame that
created the dangling reference is long gone from the stack; the last test to start is where the
collector happened to run, not necessarily the culprit.

Last three tests started before the fault:

```
PinWright.blueprint.scs.add_component.MaterialTraversalIsSecurityViolation
PinWright.blueprint.scs.add_component.MeshPreflight
PinWright.blueprint.scs.add_component.NoMaterialIsUnchanged   <- crash ~18 s in
```

**Run B — the same fourteen groups with `PinWright.blueprint` removed.** `COMPLETED_WITH_SKIPS`.

```
found=2097 started=2097 succeeded=2097 failed=0 skipped=2 performed=2097
drainMarker=True testExit=True
```

**Run C — `PinWright.blueprint+PinWright.Blueprint` alone.** `COMPLETED_CLEAN`.

```
found=194 started=194 succeeded=194 failed=0 skipped=0 performed=194
drainMarker=True testExit=True
```

## What the three runs establish

- The blueprint tests are **not** the defect: 194/194 clean in isolation, including the exact
  `scs.add_component.NoMaterialIsUnchanged` the combined run died on.
- `TestSCSAddComponentMaterialStrict.cpp`, which owns that test, was **not touched** by the
  `548bf740..HEAD` range, so this is not obviously a regression from that batch. It was not
  measured before the batch either, so it cannot be called pre-existing with confidence — see the
  baseline gap below.
- The trigger is **accumulated state across ~300 preceding tests from other groups**. Run B proves
  those 2097 tests are individually fine; run C proves the blueprint group is fine; only the
  combination fails.

## Why this matters beyond one crashed run

The project's guidance is to run tests **scoped to the groups a change touches, in one editor
instance**, precisely to avoid one-instance-per-group cost. This defect makes that instruction
unsafe at scale: the more groups you legitimately combine, the likelier the host dies, and the
failure presents as *"304 succeeded, 0 failed"*. A caller who checks `Result={Fail}` — which is the
obvious thing to grep, and what a natural reading of the run instructions suggests — sees a clean
run and concludes the suite passed. Only the drain marker distinguishes them, which is exactly what
`check_suite_log.py` exists for and exactly why it must never be skipped.

`B-suite-log-completeness-unverifiable` is adjacent but different: that ticket is about whether the
log can be trusted to be complete; this one is a reproducible way to *make* it incomplete.

## No baseline exists to compare against

`b79ba53e` records a 4576-test measurement, and its own entry states the measured tree was
`efe24958` with uncommitted `Tests/Render/` edits from a parallel session, "none of which were built
or measured here — this is not a reading of HEAD". So there is no prior clean multi-group run of
this tree to say whether the crash is new. Establishing one is the first step, not a fix.

## Suggested next step

Bisect by halving the group list rather than by test: run the first seven groups plus
`PinWright.blueprint`, then the second seven plus `PinWright.blueprint`, and narrow to the group
whose presence makes blueprint fatal. That names the leaking group in ~3 runs without needing to
identify the leaking object. Once the pair is known, `-gcdebug` / `gc.CollectGarbageEveryFrame 1`
on that pair should surface the dangling reference near where it is created rather than in a
worker-thread collection minutes later.

severity rationale: impact=a scoped suite run, which is the project's prescribed way to test a
change, silently fails to complete while presenting as 0-failures; the only tell is a missing
marker x reach=any run combining enough groups, i.e. the normal case for a change touching more
than one area -> High

## History
- `#1-initial-repro` `OPEN` verifier — 2026-08-28, UE 5.8, PinWright rebuilt at `b79ba53e`. Found
  while running the scoped suite to verify the `548bf740..HEAD` fix batch. Three runs as recorded
  above: combined fifteen groups `DID_NOT_COMPLETE` with a GC access violation on a background
  worker at 305/2335; the same set minus `PinWright.blueprint` `COMPLETED_WITH_SKIPS` at 2097/2097
  with 0 failures; `PinWright.blueprint` alone `COMPLETED_CLEAN` at 194/194. Not bisected further —
  recorded so the next agent does not re-derive the three-run isolation. Logs retained at
  `Saved/PinWright/test-runs/528eb171a19441489136c26e55bf1674/automation.log` (run A),
  `Saved/PinWright/test-runs/verify-changed/automation.log` (run B) and
  `Saved/PinWright/test-runs/verify-blueprint/automation.log` (run C).
