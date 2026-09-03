---
id: B-pcg-generated-graph-output-empty-after-generate
title: "pcg.generate reports pointCount 0 for a generation that produced instances (GetGeneratedGraphOutput is empty on 5.8)"
status: DONE
severity: Medium
category: bug
tags: [pcg, silent-wrong-data, readback, ue58]
encounters: 1
lastSeen: 2026-08-13T06:30:00Z
---

# pcg.generate reports pointCount 0 for a generation that produced instances

`pcg.generate` completes its ticket successfully with `completionSignal:"generated_delegate"` and
then reports `pointCount: 0, dataCount: 0` for passes that demonstrably produced geometry. A caller
reading `pointCount` to decide whether a graph did anything gets a confident, well-formed zero.

## Evidence (UE 5.8, verified 2026-08-13)

Two actors in a scratch level, both generated through `pcg.generate`:

| actor | graph | reported `pointCount` | actual ISM instances |
|---|---|---|---|
| `PW_PCGProbe` (StaticMeshActor, component added by the verb) | `PCG_CreatePointsGlobal` then `PCG_CreatePointsSphere` | `0` | **97** (`ISM_PCG_Cube_1`) |
| `PW_PCGGrid` (`BP_PCG_Grid`) | `PCG_CreatePointsGrid_BP` | `0` | **64** (`ISM_PCG_Cube_0`) |

Both components report `generated == True`. Instance counts read via Python
(`get_components_by_class(unreal.InstancedStaticMeshComponent)` → `get_instance_count()`), which is
independent of any PinWright code.

## Root cause — CONFIRMED against PCG source (UE 5.8), and the original hypothesis was wrong

The hypothesis below the line was that PCG *discards* the output after execution unless some
retention flag is set. **That is not what happens.** There is no retention flag, nothing is
discarded, and the value is not delegate-scoped.

`UPCGComponent::PostProcessGraph` (`C:\UE_5.8\Engine\Plugins\PCG\Source\PCG\Private\PCGComponent.cpp:627`):

- `:637` `ClearGraphGeneratedOutput();` — reset at the start
- `:660` `for (const FPCGTaggedData& TaggedData : Context->InputData.TaggedData)`
- `:750` `GeneratedGraphOutput.TaggedData.Add(OutputTaggedData);`

`GeneratedGraphOutput` is populated **exclusively from `Context->InputData`**, i.e. only from the
data that reached the **graph's Output node**. And it is *not* cleared afterwards: the fill at `:750`
precedes the broadcast at `:803`, and `:823` reads the collection back after the broadcast. So
capturing inside `OnPCGGraphGeneratedDelegate` gains nothing — the delegate sees exactly what
persists.

**`pointCount` therefore never meant "points the graph produced". It meant "points that reached the
graph's Output node".** The probe graphs end in a spawner whose `Out` pin is not wired to the graph
Output node, so zero tagged data is the *correct* engine answer — while 97 and 64 instances were
spawned. The verb was reporting an honest answer to a question nobody asked, in a field whose name
implied the other question.

Partitioned components are empty for a *second, independent* reason
(`Private/Subsystems/PCGSubsystem.cpp:748-753`): local generation tasks are deliberately kept out of
the original component's data dependencies ("those resources should be managed locally"), and `:844`
routes them to `ExecutionDependencyTasks`. Only an Unbounded-grid slice (`:767`, `:771-777`) ever
lands on the original.

<details><summary>Original hypothesis (superseded)</summary>

The count comes from `PinWrightPCG::CountGeneratedPoints(Comp->GetGeneratedGraphOutput())`
(`Source/PinWrightPCG/Private/Handlers/PCG/PCGGenerateHandler.cpp:227`). On UE 5.8
`UPCGComponent::GetGeneratedGraphOutput()` appears to be **empty once generation has completed** —
PCG does not retain the output `FPCGDataCollection` past execution unless inspection/debug retention
is enabled. The spawned ISM/HISM components persist; the point data does not.

</details>

## Not a regression

This is pre-existing, not introduced by `a4a44281` (the ticketing rewrite). The arithmetic is
byte-identical across that commit: old `PCGGenerateHandler.cpp:140` → new `:227`. `a4a44281` added
only two new reads of the same accessor, both inside the new watchdog.

## Second symptom from the same cause

`pcg.generate {force:false}` on an already-generated component returns
`PCG_GENERATION_NOT_SCHEDULED` with the message *"nothing was generated and no prior output exists"* —
a claim that is flatly false when 64 instances are sitting on the actor. The inline-readback branch
(`PCGGenerateHandler.cpp:333`, pre-existing, old `:210`) gates on the same empty accessor.

## Third symptom — latent risk in the new watchdog

The watchdog added by `a4a44281` (`:425`, `:451`) uses `GetGeneratedGraphOutput().TaggedData.Num() > 0`
as its "did this produce output" signal. It was never reached in the verification runs (the generated
delegate always resolved first), but on a `state_poll` path it would read "no output" for a pass that
produced instances. It resolves ambiguity in favour of success by design, so the blast radius is
bounded — but it is built on a signal that is empty on 5.8.

## Suggested direction

Count what survives generation rather than what does not: `UPCGComponent`'s generated resources
(`GetGeneratedResources` / the managed ISM components and their instance counts), or opt into output
retention explicitly if PCG exposes a supported flag. Whatever replaces it, the response should not be
able to report `0` for a pass that spawned instances. Failing that, drop the field rather than serve a
confident wrong number.

## Related

- `B-pcg-generate-deadlocks-game-thread` — the ticketing fix; blocked from `DONE` by this defect.

## History
- `#1-found-during-integration-verification` `OPEN` tester — Found while runtime-verifying the
  `pcg.generate` ticketing fix. The ticket lifecycle itself works correctly; this is a separate,
  pre-existing readback defect that prevented that ticket's `pointCount > 0` acceptance criterion from
  ever being satisfiable.
- `#2-honest-counts-from-managed-resources` `IN-REVIEW` developer — Root cause confirmed against PCG
  source and the filed hypothesis corrected: nothing is discarded and there is no retention flag;
  `GeneratedGraphOutput` only ever holds what reached the graph's **Output node**
  (`PCGComponent.cpp:637/:660/:750`), and it survives the completion broadcast (`:803`, read back at
  `:823`), so the delegate-capture idea is a dead end. A spawner graph with an unwired `Out` pin
  legitimately yields zero there. Fixed by counting what survives instead:
  `PCGGenerateReadback.h` gained `FGenerationReadback` / `ReadGeneration` /
  `AccumulateManagedResourceCounts` / `HasProducedOutput` / `BuildGenerationPayload`, summing
  ISM/HISM instances via `ForEachManagedResource` + `Cast<UPCGManagedISMComponent>` +
  `GetComponent()->GetInstanceCount()` (the engine's own idiom, `PCGStaticMeshSpawnerTest.cpp:183`;
  present on 5.3-5.8, no version gate needed) plus `UPCGManagedActors` for `spawnedActorCount`, and
  walking local partition components via `ForAllRegisteredLocalComponents` on 5.7+.
  `PCGGenerateHandler.cpp` now **omits** `pointCount`/`dataCount` unless `graphOutputAvailable`,
  reports `instanceCount`/`instancedComponentCount`/`spawnedActorCount` behind an always-present
  `resourceCountsAvailable`, and explains every omission in `warnings[]` (house convention: plain
  strings, set only when non-empty). The second symptom is fixed too — the `force:false` inline
  branch and both watchdog gates now use `HasProducedOutput` instead of
  `GetGeneratedGraphOutput().TaggedData.Num() > 0`, so the false "no prior output exists" message is
  gone. Two tests added in `TestPCGGenerateHandler.cpp`
  (`CountsSpawnedInstancesNotBareZero`, `PayloadReportsRealPointCount`).
  **NOT yet built or run** — authored under a no-build/no-editor constraint; needs the runtime
  verification listed in the scratchpad note before this can move to DONE.
- `#3-built-and-verified-live` `DONE` tester — Built clean on UE 5.8 with `-DisableAdaptiveUnity`
  (unity merging in force, so the code is proven under the merge, not just standalone); zero compile
  errors and zero warnings. Full suite 3605/3607, the only 2 failures being the pre-existing
  `localization.Validation.*` pair (missing `Config/Localization/Game_Gather.ini`) — baseline was
  3588/3590, and the delta is exactly the 17 tests these two workstreams added. Both new PCG tests
  pass, and `CountsSpawnedInstancesNotBareZero` passed with an EMPTY event block, so the ISM fixture
  really held instances rather than degrading to its skip path. Live repro re-run: `pcg.generate` on
  a fresh actor with `PCG_CreatePointsSphere` returned `graphOutputAvailable: false`, **no
  `pointCount` key**, `instanceCount: 97`, `instancedComponentCount: 1`, and the `warnings[]` entry
  naming the Output node. The 97 was cross-checked independently through Python
  (`get_components_by_class(InstancedStaticMeshComponent)` → `get_instance_count()` = 97, 1 component)
  — an exact match, so the readback measures reality rather than restating its own arithmetic. Second
  symptom confirmed fixed: `force:false` on the already-generated component returned
  `completionSignal: "already_generated"`, not `PCG_GENERATION_NOT_SCHEDULED`. Mechanism proven rather
  than merely worked around: `pcg.inspect` on `PCG_CreatePointsSphere` shows edges
  `CreatePointsSphere_1.Out -> StaticMeshSpawner_1.In` and three property edges, and **no edge
  whatsoever terminating at `DefaultOutputNode`** — so the empty `GetGeneratedGraphOutput()` is
  correct engine behaviour, exactly as `#2` argued. A second graph (`PCG_CreatePointsGrid_BP`)
  reported `instanceCount: 8`, also an exact match to the measured ISM count; the ticket's historical
  64 was bounds-dependent on the original actor, and matching the measurement is the stronger check.
  Committed as `6f60b0c0`.
