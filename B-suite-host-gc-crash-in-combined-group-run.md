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

## Bisection attempt: did not reproduce, and the premise does not hold

Four runs. **None reproduced the fault**, including a re-run of run A's configuration in run A's
own host project with run A's own plugin binary, which registered the identical 2335 tests.

| run | host project | filter | GC elimination | verdict |
|---|---|---|---|---|
| 1 | `PDS.uproject` | `actor+blueprint` (343) | off (project default) | `COMPLETED_CLEAN` 343/343 |
| 2 | `PDS.uproject` | run A's fifteen groups (2338) | off (project default) | drained 2338/2338, 2 unrelated failures |
| 3 | `PDS.uproject` | run A's fifteen groups (2338) | **forced on** (`-EnableGarbageElimination`) | drained 2338/2338, same 2 failures |
| 4 | `EAContentExamples58.uproject`, run A's DLL | run A's fifteen groups (2335) | on (engine default) | `COMPLETED_WITH_SKIPS` 2335/2335, **0 failures** |

Logs: `<scratchpad>\gcbisect-1.log`, `gcbisect-2.log`, `gcbisect-3.log`, `gcbisect-4-eacontent.log`.
The two failures in runs 2 and 3 are `PinWright.infra.declared_params.HandlersOnlyReadDeclaredParams`
and `PinWright.infra.wiki_src.SourcePagesFollowRenderingRules` — source-tree lint tests, failing
because ~20 agents were mid-edit on that tree. Not GC-related.

## The suggested halving bisect could not have worked

Automation orders the queue **alphabetically by full test path, case-insensitively** — not by
filter order. Reconstructing run A's selection from a full-suite log reproduces `found=2335`
exactly and puts these at ordinals 303/304/305, the same three tests run A names as its last
started:

```
303 PinWright.blueprint.scs.add_component.MaterialTraversalIsSecurityViolation
304 PinWright.blueprint.scs.add_component.MeshPreflight
305 PinWright.blueprint.scs.add_component.NoMaterialIsUnchanged
```

So of the fifteen filtered groups only **two ever executed** before the fault: `actor` (149 tests
— `PinWright.actor` prefix-matches `actor_utils` too) and the first 156 of `blueprint`/`Blueprint`.
`bpir`, `container`, `infra`, `lighting`, `material`, `Model`, `niagara`, `property`, `render`,
`Sequencer`, `system`, `texture` and `widget` all sort **after** `blueprint` and never ran. The
proposed "first seven + blueprint, then the second seven + blueprint" split would have put
`blueprint` first in the second arm and measured nothing.

Run 1 tested the only pair that could be implicated — `actor+blueprint`, 343 tests, identical
ordering with the same tests at 303/304/305 — and drained clean.

## The crash is nondeterministic; there was already evidence of that

Three archived full-suite runs on this tree used the **identical** filter (`PinWright`, 4576
tests) and disagree with each other:

```
Saved/PinWright/test-runs/batch4/automation.log  DID_NOT_COMPLETE      started 4009
Saved/PinWright/test-runs/batch5/automation.log  DID_NOT_COMPLETE      started 2225
Saved/PinWright/test-runs/batch6/automation.log  COMPLETED_WITH_SKIPS  4576/4576
```

Three different stopping points for one filter is a flaky host, not a deterministic cross-group
dangling reference. **A group-halving bisect cannot converge on that**, which is why the section
above supersedes "Suggested next step".

## Two corrections to the original measurement

**Run B dropped two groups, not one.** Its recorded filter omits `PinWright.system` as well as
`PinWright.blueprint` — thirteen filters, not fourteen. 2335 − 194 (blueprint+Blueprint) − 44
(system) = 2097, exactly run B's `found`.

**Runs A/B/C were not run against this project.** They used
`X:\src\unreal\EAContentExamples58\EAContentExamples58.uproject` and that project's own plugin
copy (the 40,644,096-byte / 08:11:48 DLL the ticket names). Their logs live under
`X:\src\unreal\EAContentExamples58\Saved\PinWright\test-runs\`, not under this checkout.

## The two hosts run different garbage-collection semantics

`X:\src\unreal\unreal-fpv-new\Config\DefaultEngine.ini:420` (in place since at least 2025-11-29):

```ini
[/Script/Engine.GarbageCollectionSettings]
gc.GarbageEliminationEnabled=False
```

EAContentExamples58 sets no GC options at all and takes the engine default (**enabled**).
Runtime-confirmed: PDS logs `Set CVar [[gc.GarbageEliminationEnabled:0]]`; run A's log has no such
line.

This matters independently of this ticket. With elimination disabled, `MarkAsGarbage()` cannot
destroy an object anything still references, so the plugin's dominant test-teardown idiom —
`MarkAsGarbage()` then `CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS)` — cannot produce a stale
`UObject*`. With it enabled (the engine default) the object is torn down regardless of remaining
raw references. **PDS masks this entire bug class**, so a suite that is only ever green on PDS is
not evidence it is green elsewhere. Force the engine default with `-EnableGarbageElimination`
(a first-class engine command-line override, `ObjectBaseUtility.cpp:195-204`) when validating.
Run 3 did exactly that and still did not reproduce, so the config difference alone is not
sufficient — but it remains a real gap in what PDS can detect.

## Crash site

Not an incidental collection. Run A's game thread goes silent for **18 s** after
`LogBlueprint: Compiling Blueprint '/Game/__PW_GatewayTests/BP_ScsNoMaterial_…'` (03:27:44.288)
and the worker faults at 03:28:02.153. The collection is forced by the test's own teardown:
`Tests\Blueprint\TestSCSAddComponentMaterialStrict.cpp:57-106` (`CleanupScsMatStrictAsset`,
reached through `ON_SCOPE_EXIT`) ends both branches with `MarkAsGarbage()` +
`CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS)` at lines 81 and 104. The blueprint group is the
**detector**, not the cause — it is where a forced collection first walks the graph after the
actor group has run. Siblings do the same: `TestSCSDuplicateComponentHandler.cpp:55,80`,
`TestSCSGetLocalChildParentLink.cpp:61`, `TestSCSSetSplinePoints.cpp:63`,
`TestBlueprintListExcludesInMemoryInstances.cpp:104`.

`EXCEPTION_ACCESS_VIOLATION reading 0x0c` is `UObjectBase::InternalIndex` (offset 0xC) off a
destroyed base — consistent with the reachability batcher walking a freed `UObject`.

## Where a dangling reference could originate (source review; not observed)

There is **no `FGCObject` subclass, no `AddReferencedObjects` override and no
`Collector.AddReferencedObject` anywhere in `Plugins\PinWright\Source`**, so the stale reference
is not held by a plugin-owned GC root.

Strongest structural candidate in the actor group:
`Tests\Actor\TestSpawnMaterialSurvivesConstructionScript.cpp:145-167` (`DiscardFixtureBlueprint`)
force-marks a compiled Blueprint, clears its keep-alive flags, marks its **`UPackage`** garbage and
collects — in the one actor test that first spawns a live instance of that class into the open
persistent level and then reinstances it (`RenameComponentMemberVariable` + `CompileBlueprint`).
Anything still holding the pre-compile instance survives the collect with its class's package
purged.

Secondary: `Tests\World\TestActorDuplicateMeshIntegrity.cpp:94-101` and `:134-138` are the only
actor-group tests spawning into the live editor world without `FScopedEditorWorldActorGuard`,
tearing down with a bare `AActor::Destroy()` — skipping the deselect that
`Tests\TestWorldUtils.h:93-99` documents as mandatory.

Latent, unrelated to this repro, same shape, worth its own ticket:
`Handlers\Niagara\NiagaraSystemViewModelCache.cpp:20-24` pins `FNiagaraSystemViewModel` (an engine
`FGCObject`) in a process-lifetime `TMap` with weak keys, so a pinned view model whose system was
collected keeps reporting references from `AddReferencedObjects`.

## What the next investigator should do

Stop bisecting by group; a flaky crash will not bisect. Instead:

1. Loop the full suite N times on a host with `-EnableGarbageElimination` and record the stop point
   each time. Crash location is the dependent variable, not a constant to bisect toward.
2. Run `gc.CollectGarbageEveryFrame 1` on `actor` **alone** (149 tests) — that moves the detector
   into the suspect group instead of waiting for the blueprint group's forced collect.
3. Instrument rather than infer: `gc.AllowParallelGC 0` moves the fault onto the game thread, where
   the stack names the referencing object instead of a worker.
4. Keep `check_suite_log.py` mandatory regardless of cause. The false-green shape it catches is
   real and is the actual operational risk.

severity rationale: impact=a scoped suite run, which is the project's prescribed way to test a
change, silently fails to complete while presenting as 0-failures; the only tell is a missing
marker x reach=any run combining enough groups, i.e. the normal case for a change touching more
than one area -> High

## The GC-elimination hypothesis is refuted — two independent lines of evidence

The standing explanation ("PDS sets `gc.GarbageEliminationEnabled=False`, which masks this whole
bug class") does not survive either an engine-semantics reading or PDS's own crash archive.

### 1. The `MarkAsGarbage` teardown idiom cannot plant a stale `UObject*` in GC-visible storage — under EITHER setting

UE 5.8 reachability handles every GC-visible reference to a garbage-marked object. The two
settings differ only in which of two SAFE outcomes you get:

- **Killable (reflected) references** — UPROPERTY / `TObjectPtr` / `AddReferencedObject`, reached
  through `ProcessReferenceDirectly` and `HandleBatchedReference(FResolvedMutableReference)`
  (`GarbageCollection.cpp:3055-3097`). With elimination **on**, the reference is `KillReference`d
  — set to null — and does **not** mark the object reachable. With it **off**, `MayKill` (`:2997`)
  returns `EKillable::No`, the reference marks the object reachable, and the object **survives**.
  Nulled or alive; never dangling.
- **Immutable references** — `Class`, `Outer` and `ExternalPackage`, emitted via
  `HandleImmutableReference` (`FastReferenceCollector.h:1018-1023`), which passes
  `bAllowReferenceElimination=false` (`:799-802`). These are never killed and **always** mark the
  referent reachable, *including when it is garbage*.

The second bullet directly kills the ticket's "strongest structural candidate". A
`UPackage->MarkAsGarbage()` whose child object survives is kept alive **by that child's own Outer
reference**. A surviving `UBlueprintGeneratedClass` can never end up "with its class's package
purged" — that state is structurally unreachable. `DiscardFixtureBlueprint` cannot produce it.

That candidate is also already guarded in source, independently.
`Tests\Actor\TestSpawnMaterialSurvivesConstructionScript.cpp:238-241` declares
`ON_SCOPE_EXIT { DiscardFixtureBlueprint(BPPath); }` **before** `FScopedEditorWorldActorGuard
WorldGuard`, with the comment "Declaration order matters: ON_SCOPE_EXIT runs LAST, the world
guard's destructor FIRST, so the spawned instance is destroyed before its Blueprint class is
discarded." The live instance is gone before the mark.

Raw (non-reflected) C++ pointers are invisible to GC under both settings, so elimination changes
nothing about them — and a dangling raw pointer faults at its *use* site, not inside
`DrainValidatedFull`. The original note's "the object is torn down regardless of remaining raw
references" conflates the two: elimination governs *reflected* references only, and it makes them
safe by nulling them.

### 2. No GC-walk fault has ever been recorded on this host

`X:\src\unreal\unreal-fpv-new\Saved\Crashes\` holds **71 non-ensure crash reports** spanning
June–August 2026. **Zero** mention `DrainValidated`, `CollectReferencesForGC`, `TReferenceBatcher`
or `FastReferenceCollector`. `gc.GarbageEliminationEnabled=False` is not what has been protecting
PDS from this fault; no such fault has ever occurred here to be protected from.

## batch4 and batch5 are not evidence of a crash

History `#2` rests on "batch4/5/6 show one identical full-suite filter stopping at 4009, 2225 and
4576 — the crash is nondeterministic". batch4 and batch5 do not support that reading.

Both logs end on a **complete, newline-terminated line**. Neither contains
`EXCEPTION_ACCESS_VIOLATION`, `Crash in runnable thread`, `RequestExitWithStatus` or any `Fatal`.
Both end on the **same two-line sequence** — `LogUObjectGlobals: Force Deleting 1 Package(s)`
followed by a `LogDatasmithContent` deprecation warning — i.e. both were inside
`ObjectTools::ForceDeleteObjects` when writing stopped (batch4 on
`/Game/PinWrightTests/PA_BarePhys_...`, a PhysicsAsset, during
`PinWright.skeleton.create_physics_asset.FromBareSkeleton`; batch5 on
`/Game/GeneratedMeshes/PW_AssetCreate_...`, a StaticMesh, during
`PinWright.Geometry.AssetCreate.ReferencedForeignSourceAssetRebuildsInPlaceWithOverwrite`).

And **neither produced a crash report**. The newest non-ensure report in `Saved\Crashes\` predates
both runs (2026-08-27 20:39, an unrelated `TArray` aliasing assert). The three reports that DO
fall inside the batch4/batch5 windows are all `IsEnsure=true` and non-fatal by construction:
`RegisterMorphTarget: Jaw_Open has empty data` (`SkeletalMesh.cpp:4855`, reached from
`MorphTargetHandler.cpp:154`) and two `_test.no_response_guard` dispatcher ensures
(`RpcDispatcher.cpp:245`).

So batch4/5 are terminations with **no crash artifact of any kind** — the
`B-suite-log-completeness-unverifiable` class, not this ticket's fault. Three stopping points for
one filter remains a fact; attributing two of them to a GC access violation is not supported by
the logs, and that attribution is load-bearing for the "nondeterministic host" conclusion.

## Teardown audit: what was inventoried, and what is exposed

**Inventory.** 75 real `MarkAsGarbage()` call sites across 27 files (81 raw `MarkAsGarbage`
matches, six of which are prose in comments). 32 sites in `Tests\` across 11 files; 43 in
handler/runtime code across 16 files. 13 sites mark a `UPackage`.

18 of the 32 test-side sites are the "mark, then force a collect" helpers this ticket names:
`TestSCSAddComponentMaterialStrict.cpp:77,80,97,102`, `TestSCSDuplicateComponentHandler.cpp:51,54,73,78`,
`TestSCSGetLocalChildParentLink.cpp:53,59`, `TestSCSSetSplinePoints.cpp:55,61`,
`TestSpawnMaterialSurvivesConstructionScript.cpp:163,165`, `TestAssetSearchNativeSubclassLive.cpp:125,131`,
`TestBlueprintListExcludesInMemoryInstances.cpp:100,103`. The other 14 mark without a forced
collect and let the next GC take the object — the safe form.

**Exposed sites: zero, by the criterion in question.** In all seven helpers
`CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS)` is the **last statement**; no pointer is read after
it and every marked object is a function-local that dies with the frame. All seven call
`RemoveFromRoot()` before `MarkAsGarbage()`, which additionally avoids the engine's `Fatal` for a
root-set object marked garbage (`GarbageCollection.cpp:4412`). Confirming the ticket's own note:
no plugin-owned `FGCObject`, no `AddReferencedObjects` override and no
`Collector.AddReferencedObject` exists anywhere in `Plugins\PinWright\Source`.

**The one behavioural difference elimination does introduce is not a crash.** With it on, a
UPROPERTY on a *surviving* object that points at a marked-garbage object is nulled;
`UBlueprintGeneratedClass::ClassGeneratedBy` is the plausible instance in these fixtures. Editor
code that assumes it non-null would null-deref **at that use** — a different fault, on the game
thread, with a stack that names the referencing object. Not `DrainValidatedFull`.

## Real defects the audit did surface (none explains the GC signature)

1. **11 test files spawn into the live editor world via `SpawnActorInActiveWorld` and never use
   `FScopedEditorWorldActorGuard`**: `Tests\Environment\TestEnvironmentDirtyFlags.cpp`,
   `Tests\Sequencer\TestSequencerAddActorBinding.cpp`, `...CameraRigBinding.cpp`,
   `...ControlRigTrack.cpp`, `...FbxRoundtrip.cpp`, `...RemoveActorsNameLabel.cpp`,
   `...SectionRangeUnits.cpp`, `Tests\World\TestActorDuplicateComponentHandler.cpp`,
   `Tests\World\TestActorDuplicateMeshIntegrity.cpp`, `Tests\World\TestCreateProceduralTerrainLabel.cpp`,
   `Tests\World\TestGetComponentsLargePayload.cpp`. The guard's destructor
   (`Tests\TestWorldUtils.h:78-125`) documents its deselect as mandatory and names the fatal it
   prevents ("Element type ID '0' has not been registered!"), and restores the persistent level's
   dirty flag. A bare `Actor->Destroy()` in an `ON_SCOPE_EXIT` does neither. Correction to the
   earlier note: `TestActorDuplicateMeshIntegrity.cpp` is **not** the only unguarded actor-group
   case — `TestActorDuplicateComponentHandler.cpp` is a second one in the same group.
2. **Five Niagara test files root a `UNiagaraSystem` fixture with no matching `RemoveFromRoot`**:
   `TestNiagaraResetModuleInput.cpp:202`, `TestNiagaraMoveModule.cpp:122`,
   `TestNiagaraGetModuleInputs.cpp:54`, `TestNIRGraphLinkCoverage.cpp:62`,
   `TestNIRGraphDataflow.cpp:98`. Sibling files pair the same `AddToRoot` with an
   `FAuthorableSystemRoots` guard. Same shape in `Tests\Assets\AnimAuthoringTestFixtures.h:42,80`.
   Root-set growth is monotonic for the process, so it only shows in long combined runs. Small in
   absolute terms (a handful of objects per run), so it is hygiene, not the fault.
3. **`ObjectTools::ForceDeleteObjects` is the highest-value follow-up.** It runs 535 / 331 / 599
   times in batch4 / batch5 / batch6 and is exactly where both PDS truncations stopped. The plugin
   already documents it as the hazard (`Tests\TestAssetTeardown.h:32`,
   `Tests\World\TestLevelHandlers.cpp:321`) and already ships the safe replacement idiom — clear
   `RF_Public|RF_Standalone`, set `RF_Transient`, `Rename` into the transient package, then collect
   — in `PwTestAssetTeardown::DiscardCreatedAssetByObjectPath` and `DiscardProbeMapPackage`.
   Migrating the remaining `CleanupTestAsset` / `DeleteAsset` callers onto it targets the API the
   evidence actually implicates, unlike the `MarkAsGarbage` idiom.

## No code was changed, and why

The premise for the fix — an unsafe teardown that PDS's config merely hides — does not hold, so
there is nothing to make unconditionally safe under this ticket. The three defects above are real
but separate, and an 11-file teardown refactor could not be compiled or run in this checkout (a
parallel wave was mid-edit), so landing it would be an unverified change of exactly the size the
project rules forbid. Status stays `OPEN`.

**No regression test was added, deliberately.** Reproducing this needs a real GC fault, which has
never occurred on this host; and a test asserting "`MarkAsGarbage` + `CollectGarbage` leaves no
dangling GC-visible reference" would only re-assert engine behaviour already guaranteed by
`HandleImmutableReference` / `KillReference`. What could not be tested: nothing was compiled or
executed — every claim above is from engine source at `C:\UE_5.8\Engine\Source`, plugin source,
archived logs under `Saved\PinWright\test-runs\`, and `Saved\Crashes\`.

**Severity note (not applied):** the fault is now unobserved on PDS across the entire crash
archive, and the two PDS data points cited for it are unattributable. That argues for a re-score,
left to re-triage rather than changed here.


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
- `#2-bisect-did-not-reproduce` `OPEN` reporter — "Not bisected to a group: four runs, none
  reproduced, including run A's exact filter in run A's own host project with run A's own DLL
  (2335/2335 drained, 0 failures). Crash site is the forced
  `CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS)` in `CleanupScsMatStrictAsset`
  (TestSCSAddComponentMaterialStrict.cpp:81,104), so the blueprint group is the detector, not the
  cause. Automation orders tests alphabetically, so only `actor` + the first 156 blueprint tests
  ever ran before the fault and the proposed halving bisect could not have worked; `actor+blueprint`
  alone drains clean. batch4/5/6 show one identical full-suite filter stopping at 4009, 2225 and
  4576 — the crash is nondeterministic, not a group combination. Also: run B dropped `system` as
  well as `blueprint` (2335-194-44=2097), runs A/B/C were on EAContentExamples58 not PDS, and PDS
  sets `gc.GarbageEliminationEnabled=False` which masks this whole bug class."
- `#3-truncated-run-observed-on-pds` `OPEN` reporter — "A full-suite PDS run on 2026-08-28 stopped at 4312 of 4625 tests with 0 failures, 0 crash markers and NO `TestExit: Automation Test Queue Empty` line — a truncated run that reads as green on a Result={Fail} count alone. Contradicts the same-day bisection note that PDS always drains: PDS can truncate too, just far later in the queue than the EAContentExamples58 runs. An immediate re-run of the identical filter drained fully (4625 performed, TestExit present), so it is nondeterministic here as well. Practical consequence: the `N tests performed` + TestExit pair is the only sound drain check; grepping for the literal 'Automation Test Queue Empty' also matches the -TestExit command-line echo and yields a false positive."
- `#4-gc-elimination-hypothesis-refuted` `OPEN` developer — "Audited the teardown idiom and REFUTED the gc.GarbageEliminationEnabled=False explanation on two independent grounds. Engine semantics: reflected references to a garbage-marked object are either KillReference'd to null (elimination on, GarbageCollection.cpp:3055-3097) or keep the object alive (off, MayKill :2997), and Class/Outer/ExternalPackage go through HandleImmutableReference with bAllowReferenceElimination=false (FastReferenceCollector.h:1018-1023, :799-802) so they are never killed and always mark the referent reachable - so a package marked garbage whose child survives is kept alive by that child's own Outer, and the 'class's package purged' candidate is structurally impossible; that test also already destroys the spawned instance first (TestSpawnMaterialSurvivesConstructionScript.cpp:238-241). Empirics: Saved/Crashes holds 71 non-ensure reports from June-August 2026 and ZERO mention DrainValidated / CollectReferencesForGC / TReferenceBatcher / FastReferenceCollector, so the fault has never occurred on PDS at all. Also corrected History #2: batch4 and batch5 carry no crash marker, no Fatal and NO crash report (the only reports in their windows are IsEnsure=true), and both end on a complete line inside ObjectTools::ForceDeleteObjects - they are the truncated-log class, not GC crashes, so two of the three 'nondeterministic' data points are unattributable. Audit numbers: 75 MarkAsGarbage() call sites in 27 files (32 in Tests/, 13 marking a UPackage); 18 test sites sit in the seven mark-then-collect helpers; EXPOSED SITES = 0 - CollectGarbage is the last statement in every helper, every marked object is a function-local, all seven RemoveFromRoot() first, and the plugin owns no FGCObject or AddReferencedObjects. Separate real defects found: 11 test files spawn into the live editor world without FScopedEditorWorldActorGuard (including a second actor-group file, TestActorDuplicateComponentHandler.cpp, which the earlier note missed); five Niagara test files plus AnimAuthoringTestFixtures.h AddToRoot fixtures with no RemoveFromRoot; and ForceDeleteObjects (535/331/599 calls per full run) is where both PDS truncations stopped, with a safe replacement idiom already shipping in PwTestAssetTeardown::DiscardCreatedAssetByObjectPath. NO CODE CHANGED - the premise for a fix does not hold and nothing could be compiled or run during the parallel wave; status stays OPEN."
