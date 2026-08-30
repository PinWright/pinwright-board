---
id: B-create-procedural-spawned-count-always-zero
title: "foliage.create_procedural reports instances_spawned: 0 while placing instances, because the counter is FFoliageInfo::GetPlacedInstanceCount() — which counts instances whose ProceduralGuid is NOT valid, i.e. exactly the non-procedural ones — so the before/after delta over a procedural spawn is structurally zero and the field can never report a success"
status: IN-REVIEW
severity: High
category: bug
tags: [foliage, create_procedural, procedural-foliage, instances_spawned, silent-false-negative, structurally-constant-field, verification-field, proceduralguid, inverted-code-comment]
encounters: 1
lastSeen: 2026-08-30T01:30:00+05:00
---

# The field that exists to prove the scatter happened is arithmetically incapable of saying so

`instances_spawned` was added by `B-foliage-create-procedural-empty-callback-noop` `#2` so that a
procedural scatter would be verifiable from the response instead of from a hardcoded flag. It is a
before/after delta of `FFoliageInfo::GetPlacedInstanceCount()` summed over every
`AInstancedFoliageActor` in the world. That function counts instances **whose `ProceduralGuid` is
not valid** — the hand-placed ones. Every instance a procedural simulation creates carries a valid
`ProceduralGuid`. So both terms of the delta are blind to precisely the instances the verb makes,
the difference is zero for any correct run, and `FMath::Max(0, ...)` removes the only value that
could ever have been non-zero.

## Measured live

Build: **21:37**, containing wave 1 (`2ba21649`) and **not** waves 2 and 3.

`foliage.create_procedural` returned `instances_spawned: 0`. `foliage.get_instances` against the
type the same call auto-created returned `count: 4`, `renderedInstanceCount: 4`, at real positions
inside the volume. **It placed 4 and reported 0.**

## Mechanism, re-derived at HEAD (`1a9e5778`)

`Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp` (registration `:1935`):

- `CountWorldFoliageInstances` lambda, `:2357-2374`, whose per-info accumulation is
  `Total += Info.GetPlacedInstanceCount();` at `:2368`
- baseline `const int32 InstancesBefore = CountWorldFoliageInstances(ProcWorld);` at `:2391`
- delta `InstancesSpawned = FMath::Max(0, CountWorldFoliageInstances(ProcWorld) - InstancesBefore);`
  at `:2436-2437`
- emitted at `:2481`

The engine function, verbatim
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:2591-2603`, declared
`InstancedFoliage.h:364`):

```cpp
int32 FFoliageInfo::GetPlacedInstanceCount() const
{
	int32 PlacedInstanceCount = 0;
	for (int32 i = 0; i < Instances.Num(); ++i)
	{
		if (!Instances[i].ProceduralGuid.IsValid())   // <- NOT valid
		{
			++PlacedInstanceCount;
		}
	}
	return PlacedInstanceCount;
}
```

"Placed" in the engine's vocabulary means *placed by hand*, as opposed to grown by the simulation.
The predicate is the exact complement of what this verb produces.

**Every instance this verb creates carries a valid guid — four hops, none of them conditional:**

| step | site |
|---|---|
| `UProceduralFoliageComponent` mints one in its constructor for every non-template instance: `ProceduralGuid = FGuid::NewGuid();` | `C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/ProceduralFoliageComponent.cpp:32` (and again on re-register, `:347`) |
| the simulation is handed it: `Params.ProceduralGuid = GetProceduralGuid();` | `:92` |
| every desired instance the tile emits is stamped: `DesiredInst->ProceduralGuid = ProceduralGuid;` | `C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/ProceduralFoliageTile.cpp:491` |
| the placement loop copies it onto the real instance: `Inst.ProceduralGuid = PotentialInstance.DesiredInstance.ProceduralGuid;` | `C:/UE_5.8/Engine/Source/Editor/FoliageEdit/Private/FoliageEdMode.cpp:1536` |

and that placement loop is the one this verb reaches, through the reflected call it makes:
`UProceduralFoliageEditorLibrary::ResimulateProceduralFoliageComponents`
(`C:/UE_5.8/Engine/Source/Editor/FoliageEdit/Private/ProceduralFoliageEditorLibrary.cpp:57-87`) ->
`FEdModeFoliage::AddInstances(...)` at `:72`.

So `instances_spawned` is **structurally zero**, not merely zero in the measured case. There is no
input, no volume, no surface and no spawner for which it can report a placement. The one way the
delta could move is a *drop* in non-procedural instances, and `FMath::Max(0, ...)` (`:2437`) clamps
that away too — so the field has exactly one reachable value.

## Two code comments assert the opposite, and both are load-bearing

- `FoliageHandler.cpp:2363-2365` picks the function on purpose and says why:
  *"`GetPlacedInstanceCount()` is the accurate placed-count (vs `Instances.Num()`), so keep it here
  rather than the raw Instances count those callers use."* The rejected alternative,
  `Instances.Num()`, is the one that would have worked.
- `FoliageHandler.cpp:2432-2435`, immediately above the assignment: *"Zero placed (e.g. no surface
  under the volume) is a truthful, distinguishable result, not a masked success."* Zero is the only
  value it can emit, so it distinguishes nothing — and the comment names the wrong-surface story
  that has now been offered three times for this verb's zero (see below).

## What the wrong number has already cost this board

`B-foliage-create-procedural-empty-callback-noop` (IN-REVIEW, High, `encounters: 3`) is where the
field lives, and two of its history entries reason from it:

- `#2` justified not asserting a positive count in its own regression test —
  `PinWright.foliage.create_procedural.ReportsInstancesSpawned` — with *"a headless world has no
  surface under the volume (0 is honest)"*. `#3` retired that explanation on evidence (four calls
  over real terrain also returned 0) and replaced it with the zero-extent volume. **Both
  explanations are now superseded by a third that subsumes them: the count could not have been
  non-zero in either world.** That test asserts only that the field is present, so it passes on this
  defect and will keep passing.
- `#4` concluded, from a positive control: *"The `resimulated:true` / `instances_spawned:0` pair was
  truthful on both fields: dispatch did happen, and zero did land."* **The second half is false.**
  It is also worth noting that the control itself could not have exercised the field: the 2,121
  instances it reports came from
  `ProceduralFoliageEditorLibrary.resimulate_procedural_foliage_volumes` driven directly, and those
  instances carry the component's `ProceduralGuid` like any others, so `instances_spawned` would
  have read 0 for them too. The 2,121 was counted by another route, which is why it was visible at
  all.

Neither entry is being challenged on its own subject. The point is narrower and it is why this is
filed rather than folded in: **the field has been read as evidence in three separate history
entries by three agents, and it carries none.**

## This is NOT the brushless-volume defect

`B-spawned-volumes-have-no-brush-geometry` (IN-REVIEW, High) argues that the zero-extent volume is
why nothing was placed. That defect is real, was separately verified **fixed** on the 21:37 build at
both entry points, and is present at HEAD: `VolumeBrushGeometry::BuildBoxBrushGeometry(Volume, Size)`
at `FoliageHandler.cpp:2351` and `Handlers/Actor/SpawnHandler.cpp:218`.

**Consequence for whoever closes that ticket, and the reason this section exists:** its `#1`
evidence block cites `instances_spawned: 0` across four calls as a symptom, and `#3` / `#4` build
the volume-has-no-extent argument on that number. A tester must **not** use `instances_spawned`
going positive as the closing criterion — it never will, brush or no brush. Close it on the brush:
`actor.get_bounding_box` returning a real extent and an identity actor scale at both entry points,
which is what its `#4` fix actually changed. Recorded here as a recommendation only; not closed by
this ticket.

## Fix

The engine already answers this question, with the guid the plugin was told it did not have. Its
own comment at `FoliageHandler.cpp:2354-2356` says the delta approach exists *"without needing the
private guid"* — but `UProceduralFoliageComponent::GetProceduralGuid()` is public and the component
is in hand at the call site:

- boolean, zero new code: `UProceduralFoliageComponent::HasSpawnedAnyInstances()`
  (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/ProceduralFoliageComponent.h:125`, body
  `ProceduralFoliageComponent.cpp:400-412`) -> `ContainsInstancesFromProceduralFoliageComponent`
  (`InstancedFoliage.cpp:3487-3504`), which matches on `ProceduralGuid ==`. This is what the
  engine's own editor UI uses to decide whether to show its "Unable to spawn instances" toast
  (`ProceduralFoliageEditorLibrary.cpp:76-84`).
- exact count: the same walk with a counter instead of an early `return true`, mirroring
  `AInstancedFoliageActor::DeleteInstancesForProceduralFoliageComponentInternal`
  (`InstancedFoliage.cpp:3438` onward, the match at `:3448`).
- or keep the delta shape and swap the accumulator at `:2368` to `Info.Instances.Num()` — the
  option the comment at `:2363-2365` explicitly declined. Correct, but weaker: it attributes any
  concurrent foliage change to this call, which is the ambiguity the guid removes.

Whichever lands, the two comments above have to change with it, and
`PinWright.foliage.create_procedural.ReportsInstancesSpawned`
(`Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp:1657-1724`) needs an assertion
that can fail — a spawn over a real surface asserting the reported count against a
`foliage.get_instances` read, which is the differential shape that would have caught this.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* carries the enumeration; this is a
variant of it rather than a member. There the deciding number was never reported. Here it is
reported, prominently, as the verb's headline verification field — and the expression behind it has
one reachable value. That is the harder version to catch, because the field looks like the fix for
exactly the problem it has: a caller who has read this verb's history knows `instances_spawned` was
added *because* `resimulated` was a hardcoded lie, and reads the replacement as the trustworthy one.
The nearest sibling in that precise shape is `B-foliage-adds-never-rebuild-the-hism-tree`, filed
from the same pass: a verification field whose value is computed from something adjacent to, but not,
the thing it names.

## Cross-links

- **`B-foliage-create-procedural-empty-callback-noop`** (IN-REVIEW, High) — owns the field. Split
  out rather than appended for the same reason `#3` of that ticket split out
  `B-spawned-volumes-have-no-brush-geometry`: different mechanism, different file region, separately
  fixable. Its `#2` test rationale and its `#4` "zero did land" conclusion are corrected above.
- **`B-spawned-volumes-have-no-brush-geometry`** (IN-REVIEW, High) — verified fixed at HEAD on the
  brush; see the section above for what its closing criterion must and must not be.
- **`B-create-procedural-ignores-scale-and-normal-fields`** (IN-REVIEW) — its `#3` return quotes
  *"`foliage_types_count` / `instances_spawned` / `skippedCount` are all correct, and the deciding
  number is [not reported]"*. `instances_spawned` should come off that list.
- **`F-resimulate-existing-foliage-volume`**, **`F-procedural-foliage-simulation-knobs-unreachable`**
  — any new verb that re-simulates a volume needs a placement count, and will hit this same trap if
  it reaches for `GetPlacedInstanceCount()` on the name.
- **`B-foliage-adds-never-rebuild-the-hism-tree`** — the pass-sibling; see § *Same shape as*.

## Severity

**High**, on the rubric's *silent wrong data on a normal path* band: the caller trusts a result that
is a lie and builds on it, and three history entries on this board demonstrate that happening. It is
a false **negative** rather than a false success, which is the milder direction — nobody ships broken
content because of it — but the cost is the same class: the verb's only self-verification is dead,
so every use of it needs a second `foliage.get_instances` call the caller has no reason to know they
need.

**Both reach modifiers declined.** No upward bump: `foliage.create_procedural` is one of six verbs
in the namespace and not an every-session call. The downward "rare edge path" bump is declined
explicitly and is the reading being rejected: this is not an edge path, it is the **only** path —
there is no input under which the field is right, and no branch of the verb that avoids it.

**Not Critical.** Nothing is corrupted and nothing is lost; the instances are placed correctly and
persist. The rubric's Critical band needs a crash or a destructive write, and this is a reporting
defect over a correct mutation.

## History
- `#1-placed-count-excludes-procedural-instances` `OPEN` reporter — Measured on the **21:37** build
  (wave 1 `2ba21649`, without waves 2/3); mechanism re-derived against HEAD `1a9e5778`.
  `foliage.create_procedural` returned `instances_spawned: 0` while `foliage.get_instances` on the
  type the same call auto-created returned `count: 4, renderedInstanceCount: 4` at real positions
  inside the volume. **Mechanism:** the counter is
  `Info.GetPlacedInstanceCount()` (`FoliageHandler.cpp:2368`, inside the
  `CountWorldFoliageInstances` lambda `:2357-2374`), differenced at `:2436-2437` against the
  baseline at `:2391` and emitted at `:2481`. `FFoliageInfo::GetPlacedInstanceCount`
  (`InstancedFoliage.cpp:2591-2603`, decl `InstancedFoliage.h:364`) counts instances whose
  `ProceduralGuid` **is not valid** — the hand-placed ones — and every instance this verb creates
  carries a valid one, stamped along an unconditional four-hop chain:
  `ProceduralFoliageComponent.cpp:32` (ctor `FGuid::NewGuid()`, again at `:347`) ->
  `:92` (`Params.ProceduralGuid = GetProceduralGuid()`) -> `ProceduralFoliageTile.cpp:491`
  (`DesiredInst->ProceduralGuid = ProceduralGuid`) -> `FoliageEdMode.cpp:1536`
  (`Inst.ProceduralGuid = PotentialInstance.DesiredInstance.ProceduralGuid`), reached from this
  verb through `ProceduralFoliageEditorLibrary.cpp:57-87` and its
  `FEdModeFoliage::AddInstances` call at `:72`. So the delta is **structurally** zero, not zero in
  this case, and `FMath::Max(0, ...)` at `:2437` clamps away the only other reachable value. **Two
  code comments inverted by this, both quoted in the body:** `:2363-2365` picks
  `GetPlacedInstanceCount()` over `Instances.Num()` as "the accurate placed-count" (the rejected
  option is the one that works), and `:2432-2435` calls zero "a truthful, distinguishable result,
  not a masked success" when zero is the only value it can emit. **Already cost the board three
  readings**, all on `B-foliage-create-procedural-empty-callback-noop`: `#2`'s "a headless world has
  no surface" test rationale, `#3`'s replacement zero-extent explanation, and `#4`'s conclusion that
  "`resimulated:true` / `instances_spawned:0` … was truthful on both fields: dispatch did happen,
  and zero did land". `#4`'s 2,121-instance positive control could not have exercised the field
  either — those instances carry the component's guid, so the counter is blind to them, and the
  2,121 was obtained by another route. **Deliberately NOT the brushless-volume defect:**
  `B-spawned-volumes-have-no-brush-geometry`'s subject is separately verified fixed at HEAD at both
  entry points (`FoliageHandler.cpp:2351`, `SpawnHandler.cpp:218`); recommendation recorded for its
  owner to close it on the brush (`actor.get_bounding_box` extent + identity scale) and **not** on
  `instances_spawned` going positive, which it never will — not closed here. **Fix:** the engine
  already answers it — `UProceduralFoliageComponent::HasSpawnedAnyInstances()`
  (`ProceduralFoliageComponent.h:125`, `ProceduralFoliageComponent.cpp:400-412` ->
  `InstancedFoliage.cpp:3487-3504`) for the boolean, the same walk with a counter (mirroring
  `:3438` onward, match at `:3448`) for the count; the guid the handler's comment at `:2354-2356`
  calls "private" is reachable via the public `GetProceduralGuid()`. **Rated High**, false-negative
  half of the silent-wrong-data band; both reach modifiers declined, the downward one explicitly
  because this is the verb's only path. Dedup: searched the board for `instances_spawned`,
  `GetPlacedInstanceCount`, `ProceduralGuid`, `create_procedural` and every `B-foliage-*` /
  `B-create-procedural-*` / `F-*procedural*` file. `B-foliage-create-procedural-empty-callback-noop`
  owns the field's introduction but not this mechanism;
  `B-create-procedural-ignores-scale-and-normal-fields` and
  `B-create-procedural-density-writes-paint-density` are the scale and density properties;
  `B-create-procedural-terrain-paints-nothing` is `landscape.create_procedural_terrain`, a different
  verb. Nothing on the board mentions `GetPlacedInstanceCount`.
- `#2-count-by-proceduralguid-not-placed-count` `IN-REVIEW` developer — "Re-verified the mechanism
  against `C:/UE_5.8/Engine/Source` before changing anything, and **the ticket has the direction
  right**: `FFoliageInfo::GetPlacedInstanceCount` (`Runtime/Foliage/Private/InstancedFoliage.cpp`,
  the `/* Get the number of placed instances */` body) increments only under
  `if (!Instances[i].ProceduralGuid.IsValid())`, and the four-hop stamping chain is unconditional
  as described (`ProceduralFoliageComponent.cpp` ctor `ProceduralGuid = FGuid::NewGuid()` ->
  `Params.ProceduralGuid = GetProceduralGuid()` -> `ProceduralFoliageTile.cpp`
  `DesiredInst->ProceduralGuid = ProceduralGuid` -> `FoliageEdMode.cpp`
  `Inst.ProceduralGuid = PotentialInstance.DesiredInstance.ProceduralGuid`). One correction to the
  ticket's line references, not to its argument: `GetProceduralGuid()` is on
  `ProceduralFoliageComponent.h:142`, not `:125` — `:125` is `HasSpawnedAnyInstances()`. Both are
  public. **Fix, in `FoliageHandler.cpp`'s `foliage.create_procedural`:** the
  `CountWorldFoliageInstances` lambda and its before/after delta are gone. In their place
  `MeasureFoliageInstancesByProceduralGuid(World, Guid, OutProcedural, OutHandPlaced)` walks
  `FFoliageInfo::Instances` across every `AInstancedFoliageActor` **once, after** the
  resimulation, and splits on the guid — matching
  `AInstancedFoliageActor::ContainsInstancesFromProceduralFoliageComponent`'s predicate rather
  than its complement. `instances_spawned` is now the count of instances carrying the spawned
  volume component's own `GetProceduralGuid()`, so it is attributable to this call and needs no
  baseline (a freshly minted guid cannot pre-exist), and no `FMath::Max` clamp. The hand-placed
  total — what the old counter was actually measuring, and which this verb produces none of — is
  published separately as `hand_placed_instances_in_world`, so neither number can be read as the
  other. **Per the house convention:** both are MEASURED after the operation, and both are
  **omitted rather than reported as 0** when nothing could be measured (no procedural component /
  no world / no guid), with a `warnings[]` line saying the fields are absent rather than zero — a
  0 now means 'nothing landed', a missing field means 'not counted'. The measured-vs-expected
  disagreement is `bResimulated && InstancesSpawned == 0`: dispatch completed but nothing carries
  the guid, which raises a `warnings[]` line naming the surfaceless-volume cause — the same
  condition the engine's editor UI raises its 'Unable to spawn instances' toast on
  (`ProceduralFoliageEditorLibrary.cpp`), which an MCP caller never sees. Deliberately **not**
  also a `UE_LOG(Warning)`: a host on the engine default `bElevateLogWarningsToErrors=true`
  promotes an automation-run log warning into a test error, and this branch is the normal outcome
  of a surfaceless fixture. Both inverted comments named in the body are replaced by the real
  mechanism. **Regression test:**
  `PinWright.foliage.create_procedural.SpawnedCountMatchesTheFoliageActors`
  (new file `Source/PinWright/Private/Tests/Environment/TestFoliageProceduralSpawnCount.cpp`)
  — spawns a 40 m static-mesh floor through the real `actor.spawn` verb in an isolated far column,
  ticks the world so the body reaches the scene-query structure, runs `foliage.create_procedural`
  over it with `tileSize` matched to the volume so every seed lands inside the brush, then reads
  ground truth by re-walking `FFoliageInfo::Instances` for the volume component's guid — never
  from the response — and asserts the reported count equals it. It also asserts the ground truth
  is positive, so a fixture that scattered nothing FAILS rather than passing vacuously (the exact
  hole that let the old field survive three readings as evidence). **It is red today**: reported 0
  vs a positive ground truth. It additionally pins `hand_placed_instances_in_world` against an
  independently recomputed non-procedural total, so a fix that merely renamed the field fails.
  The pre-existing `ReportsInstancesSpawned` keeps every assertion it had; only its rationale
  comment changed, since 'a headless world has no surface' was the superseded explanation. Ran
  `Content/Python/check_test_ids.py`: CLEAN, 4791 ids, no dot-prefix collision. **Sibling counters
  in the same response audited, none has this defect:** `foliage_types_count` counts types
  actually built; `foliage_types_requested` is the request echo under its own name;
  `skippedCount` is measured; `tile_size` / `num_unique_tiles` are read back off the spawner
  asset after the write; `tile_overlap` is read off the component and already omitted rather than
  zeroed; `foliage_types[]` rows are read back off the assets. Wider sweep: `instancesPlaced`
  (`foliage.paint`) and `instancesRemoved` (`foliage.remove`) are incremented by their own write
  loops, and `GetPlacedInstanceCount` now appears nowhere in plugin source outside comments.
  **A live scatter comparison is now possible** — the measurement the per-type simulation-property
  work said it could not make while `instances_spawned` read zero for an unrelated reason. NOT
  compiled and NOT run per instruction; the orchestrator builds and runs. `foliage.md` wiki overlay documents
  both fields, the omission rule, the no-surface cause, and that a `0` from a pre-fix build
  carries no information. **Recommendation for `B-create-procedural-ignores-scale-and-normal-fields`
  `#3`: `instances_spawned` can come off its 'all correct' list — it was not correct, and now is.**"
