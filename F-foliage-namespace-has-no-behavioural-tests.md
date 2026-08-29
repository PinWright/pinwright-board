---
id: F-foliage-namespace-has-no-behavioural-tests
title: "No foliage test asserts where a verb put anything: 14 test cases across 6 verbs, of which 6 are ValidParamsNoCrash registration smoke and the rest are input/response honesty — foliage.paint has that smoke test and nothing else, which is the same shape that let the create_grass_type defect ship a persisted broken asset"
status: IN-REVIEW
severity: Medium
category: feature
tags: [tests, foliage, coverage, behavioural-test, valid-params-no-crash, placement, differential-proof, missing-fixture, landscape]
---

# The namespace is tested for what it says, never for what it does

Counted from `IMPLEMENT_*_AUTOMATION_TEST` category strings across both plugin modules (the macro's
category string is authoritative, not the filename — `agent-conventions.md`):

| namespace | test cases |
|---|---|
| `spatial` | 55 |
| `landscape` | 32 |
| `pcg` | 18 (+7 `pcgir`) |
| **`foliage`** | **14** |

The count is the smaller half of the problem. The shape is the rest of it. All 14:

- **Registration smoke — 6.** `foliage.paint.ValidParamsNoCrash`, `foliage.remove.ValidParamsNoCrash`,
  `foliage.get_instances.ValidParamsNoCrash`, `foliage.add_type.ValidParamsNoCrash`,
  `foliage.add_instances.ValidParamsNoCrash`, `foliage.create_procedural.ValidParamsNoCrash`. Each
  posts a payload and asserts that `InvokeHandler` returned true — i.e. that a handler is registered
  for the method name. Nothing is read back; in several the response is not even inspected.
- **Input / response honesty — 6.** `add_type.MissingMeshPath`; `remove.MissingScopeErrors`,
  `remove.NonexistentPathErrors` (regressions for `E-foliage-remove-silent-edge-inputs`);
  `add_instances.LocationsOnlyUnchanged`, `ReportsIgnoredLocations`,
  `TransformsOnlyOmitsPrecedenceFields`. All assert on the *envelope*: which error came back, which
  field was echoed.
- **Touches real geometry — 2.** `get_instances.RoundTripsScaleAndRotation` writes one instance
  through the real `foliage.add_instances` against `/Engine/BasicShapes/Cube.Cube` and reads it back
  (regression for `E-foliage-get-instances-drops-scale`), and
  `create_procedural.ReportsInstancesSpawned` runs a real volume + spawner and asserts the
  `instances_spawned` field exists (regression for `B-foliage-create-procedural-empty-callback-noop`).

**Correction to this ticket's own premise, recorded rather than quietly fixed.** It was proposed as
*"all three foliage test files are input-honesty/schema; no test asserts any foliage verb places
geometry"*. The first half is wrong — foliage tests live in four files, not three, and two of them do
drive real placement. The second half survives in a narrower and more useful form: **no test asserts
where anything landed.** `RoundTripsScaleAndRotation` asserts that a scale and a rotation survive a
write/read round trip; `ReportsInstancesSpawned` asserts that a count field is present. Neither
asserts a position, a surface, an alignment, or a spacing. The two properties this pass found broken
— `foliage.paint` performs no ground projection, and `foliage.create_procedural` flattens every
plant to one scale and vertical — are both invisible to every test that exists.

## `foliage.paint` has exactly one test, and it is the smoke test

`PinWright.foliage.paint.ValidParamsNoCrash` (`PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp:1318-1338`)
is the whole of that verb's coverage. `B-foliage-paint-does-no-ground-projection` would pass it
unchanged after any fix, and passed it unchanged before anyone noticed the defect.

## The same shape shipped a broken asset one namespace over

`PinWright.landscape.create_grass_type.ValidParamsNoCrash`
(`PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp:1821-1833`) posts
`{name: "TestGrassType", meshPath: "/Game/Meshes/SM_Grass"}` and asserts only that `InvokeHandler`
returned true. `/Game/Meshes/SM_Grass` does not exist in this project, so the test actually exercises
the handler's `ASSET_NOT_FOUND` branch — it has never once reached the asset-creation code. Behind
that green test, `landscape.create_grass_type` builds its `FGrassVariety` with `AddZeroed` and
produces an asset the engine discards at two independent gates
(`B-create-grass-type-addzeroed-never-renders`). The verb reports success, saves to disk, and
renders nothing.

This is the exact failure `agent-conventions.md` warns about: *"A missing required fixture is a test
FAILURE (assert and fail) — never a `return true` skip; skips silently green the suite while covering
nothing."* Here it is worse than a skip, because nothing declares itself skipped.

**474 `ValidParamsNoCrash` test cases exist plugin-wide.** The pattern is not the problem — a cheap
registration smoke test is worth having, and most of those 474 sit alongside real behavioural tests.
The problem is a verb for which it is the *only* test, and a smoke test whose fixture does not
resolve, so it silently degrades to testing the error path.

## What to add

Per `agent-conventions.md`, each of these must call the same production symbol the handler uses (a
test re-implementing the logic stays green after a revert) and must be shown to fail against current
source — every one below does today:

1. **`foliage.paint` places where it was asked, and says whether it projected.** Currently it always
   places at the literal XYZ; assert that against a known surface so the assertion flips when
   projection lands. Pairs with `B-foliage-paint-does-no-ground-projection`.
2. **`foliage.paint` refuses or reports a location-less entry** instead of placing it at the world
   origin, and reports a non-object `locations[]` entry in `skipped[]` the way `add_instances` does.
3. **`foliage.create_procedural` applies per-type scale and alignment.** Read the generated
   `UFoliageType`'s `ScaleX/Y/Z` and `AlignToNormal` back off the saved asset and assert they match
   the request. Fails today: those fields are never written
   (`B-create-procedural-ignores-scale-and-normal-fields`).
4. **`landscape.create_grass_type` produces a variety the engine will not discard.** On the saved
   asset assert `GrassVarieties[0].GetEndCullDistance() > 0` and `AllowedDensityRange.Max > 0`. Both
   read 0 today, so this is the differential proof for the grass fix. **Use a fixture that exists**
   — `/Engine/BasicShapes/Cube.Cube`, as the two working foliage tests already do — and assert the
   response, not just that a handler answered.
5. **Sweep `ValidParamsNoCrash` fixtures.** Any smoke test whose asset path does not resolve is
   testing the not-found branch under a name that claims otherwise. `/Game/Meshes/SM_Grass` is one;
   a scan of the other 473 for unresolvable `/Game/...` paths is a mechanical, one-off job that
   would say how many more there are. This is the generalisable half of the ticket and the part that
   pays off outside `foliage`.

## Distinct from

- **`F-controlrig-test-coverage`** (IN-REVIEW, Low) and **`F-crir-coverage-test-matrix`** (DONE, Low)
  — the board's two precedents for a coverage-gap ticket, and the reason this one carries the `F-`
  prefix rather than the `E-` it was proposed under. Both are *prospective* matrices: "this surface
  should have tests". This one is filed with an escaped defect attached, which is the argument for
  rating it above their Low.
- **`B-create-grass-type-addzeroed-never-renders`** (OPEN, High) — the escape. Its item 4 above is
  the differential-proof test that ticket needs, so a fixer taking either should take both.
- **`B-foliage-paint-does-no-ground-projection`** (OPEN, High) and
  **`B-create-procedural-ignores-scale-and-normal-fields`** (OPEN, Medium) — the two defects items 1,
  2 and 3 would have caught. Each ticket carries its own required regression test; this one exists
  because the *shape* of the gap outlives all three.
- **`E-foliage-nested-input-schemas-undocumented`** (IN-REVIEW, Low) — owns the two
  `infra.wiki_handler.MethodPage.Foliage*Schema` doc tests, which are excluded from the 14 above
  because they test the wiki page, not the verb.

## Not RPC-verified

The suite was not run. Counts are from a static scan of `IMPLEMENT_*_AUTOMATION_TEST` category
strings across `Source/PinWright` and `Source/PinWrightPCG`; the claim that each `ValidParamsNoCrash`
asserts only registration is from reading the bodies of the six foliage ones plus the
`create_grass_type` one, not all 474. The `/Game/Meshes/SM_Grass` fixture is absent from this
project's content, checked on disk. A suite run would confirm — and is the right first step for
item 5 — that these tests currently pass, which is the whole point: they pass over two known defects.

severity rationale: impact=Medium — a process gap rather than a user-facing defect, which the board's two coverage precedents rate Low, raised one band here because it is not prospective: a concrete High-severity defect (`B-create-grass-type-addzeroed-never-renders`) shipped a persisted, committed, silently-inert asset behind a green test whose fixture does not even exist, and a second verb (`foliage.paint`) currently has that identical single-smoke-test coverage × reach=normal — no modifier; the gap is in one namespace plus one landscape verb, though item 5's fixture sweep generalises across all 474 `ValidParamsNoCrash` cases and a reviewer who weights that generalisation could argue High -> Medium

## History
- `#1-no-placement-assertions` `OPEN` reporter — Static scan only; the suite was not run and the editor was not running. Counted from `IMPLEMENT_*_AUTOMATION_TEST` category strings across `Source/PinWright` and `Source/PinWrightPCG`: foliage 14 test cases, pcg 18 (+7 pcgir), landscape 32, spatial 55. Of the 14 foliage cases, 6 are `ValidParamsNoCrash` registration smoke, 6 are input/response honesty, and 2 touch real geometry. Corrects this ticket's own proposed premise rather than restating it: the claim was "3 foliage test files, all input-honesty/schema, no test asserts any foliage verb places geometry" — foliage tests actually live in 4 files, and `get_instances.RoundTripsScaleAndRotation` and `create_procedural.ReportsInstancesSpawned` do drive real placement through the real handlers. The surviving and sharper claim: no foliage test asserts a POSITION, surface, alignment or spacing — the first asserts a scale/rotation round trip, the second asserts a count field's presence — so both defects this pass found (`foliage.paint` does no ground projection; `create_procedural` flattens scale and alignment) are invisible to every existing test. `foliage.paint`'s entire coverage is one `ValidParamsNoCrash` (`Tests/World/TestEnvironmentHandlers.cpp:1318-1338`). The same shape one namespace over is how `B-create-grass-type-addzeroed-never-renders` shipped: `landscape.create_grass_type.ValidParamsNoCrash` (`:1821-1833`) uses `meshPath: "/Game/Meshes/SM_Grass"`, which does not exist in this project, so it has only ever exercised the `ASSET_NOT_FOUND` branch — a fixture failure that `agent-conventions.md` says must be a test FAILURE, silently greening instead. 474 `ValidParamsNoCrash` cases exist plugin-wide; the pattern itself is fine and the defect is a verb for which it is the ONLY test, plus smoke tests whose fixtures do not resolve — item 5 proposes a one-off scan of all 474 for unresolvable `/Game/...` paths, which is the generalisable half. Filed under `F-` rather than the proposed `E-` to match the board's two coverage-gap precedents, `F-controlrig-test-coverage` (IN-REVIEW, Low) and `F-crir-coverage-test-matrix` (DONE, Low); rated above their Low because both of those are prospective matrices and this one is filed with an escaped High defect attached. Dedup: searched the board for `test coverage`, `behavioural test`, `ValidParamsNoCrash`, `no test asserts`, `foliage` + tests, and every `F-*coverage*` / `F-*test*` file. No ticket covers foliage test coverage or the `ValidParamsNoCrash` fixture problem.
- `#2-behavioural-coverage-added` `IN-REVIEW` fixer — Added `Source/PinWright/Private/Tests/Environment/TestFoliagePlacementBehaviour.cpp`: **7 test cases across 5 of the 6 verbs**, every one asserting observed engine state. Ground truth is `FFoliageInfo::Instances` gathered by `TActorIterator<AInstancedFoliageActor>` over every IFA in the editor world — never the response, and never the actor `foliageActorPath` names (checking a handler against the actor it named proves nothing); the response is read only to cross-check its counts AGAINST that store, which is the assertion that catches a verb reporting N placed while N-1 landed. Each test builds its own `UFoliageType` through the real `foliage.add_type` under a GUID-suffixed name so its store starts empty by construction and absolute counts mean something, and tears it down with a TYPE-SCOPED `foliage.remove` (never `removeAll`, which the two pre-existing behavioural tests use and which wipes the host map's own foliage) plus `CleanupTestAsset`, so no dirty `/Game` package survives for a save-all to flush (`B-tests-leak-host-content`). Requests route through the real `FRpcDispatcher::ProcessRequest`, so payloads are also ParamSpec-validated. **The tests, each with the implementation bug that turns it red:** (1) `foliage.paint.PaintedInstancesReachTheFoliageActor` — 3 painted locations produce exactly 3 instances at the 3 requested XYs at unit scale; red if the placement loop's unconditional counter increments over the branch where `AddFoliageType`+`FindInfo` stores nothing, if entries collapse onto one point or the origin, or if `DrawScale3D` is left zero (the grass `AddZeroed` shape — an instance that renders nothing). (2) `foliage.paint.NonObjectLocationEntriesPlaceNothing` — a batch of 2 usable + 3 non-object entries stores exactly 2, none at the origin, `instancesPlaced` == store count; red if the object-type guard is dropped (junk defaults to a world-origin placement) or if the count echoes the input array's length. (3) `foliage.add_type.WritesRequestedFieldsToTheAsset` — mesh, `Density`, `ScaleX/Y/Z.Min/Max`, `AlignToNormal`, `RandomYaw` read back off the asset at the caller-constructible path, and `asset_path` resolves to that same object. **Every requested value deliberately differs from the `UFoliageType` CDO default** (`InstancedFoliage.cpp:579-590`: Density=100, AlignToNormal=true, RandomYaw=true, Scale=[1,1]), so this is exactly the differential the `create_grass_type` escape needed: a handler that creates and saves the asset and writes nothing fails every line, where asserting the defaults would have certified the gap. (4) `foliage.add_type.RefusedInputCreatesNoAsset` — `minScale > maxScale` is `INVALID_ARGUMENT` AND leaves no object and no registry entry at the path; red if a validator is ever moved below `CreatePackage`/`NewObject`, which leaves a half-built dirty package behind. (5) `foliage.remove.ScopedRemovalSparesOtherTypes` — two populated types, a scoped remove of one: `instancesRemoved`==3, `mode`=="type", the named type's store is empty and the other type's 2 instances are still at their exact placed positions; red if a scoped call falls through to the removeAll branch, or if the count is computed without the array actually being emptied. Response `mode` alone could never distinguish these. (6) `foliage.add_instances.OnlyParsableEntriesReachTheStore` — 4 transforms (full object form with non-zero rotation + non-uniform scale, a location-less entry, an array-form `[x,y,z]` location, a non-object) store exactly 2, nothing at the origin, `instances_count`==store count, `skippedCount`==2, and the stored rotation/scale are byte-exact; red if the location-less `continue` is dropped (origin placement), if either parse branch is lost, or if scale/rotation are dropped — proven at the STORE, so a symmetric write+read loss cannot hide it the way the existing round-trip test can. (7) `foliage.get_instances.FilterScopeMatchesTheStore` — filtered read returns exactly the named type's 2 rows and no others; unfiltered read reports both fixtures with `count`==array length and all 9 transform numbers matching the store; red if the filter is ignored, if the `ForEachFoliageInfo` lambda ever returns false (silent truncation after the first type, which reads as an ordinary success), or if the unfiltered branch drops rotation/scale again. **Left uncovered, deliberately, with reasons stated in the file header rather than silently:** `foliage.create_procedural` entirely — `B-create-procedural-ignores-scale-and-normal-fields` (OPEN) rewrites the exact three lines that build each `UFoliageType`, and it is already the one foliage verb with a test driving real placement; `foliage.paint`'s Z and rotation — `B-foliage-paint-does-no-ground-projection` (OPEN/High) owns them and a projection fix moves Z, so the test pins XY and scale, which survive a downward projection; `foliage.paint`'s `{}` location entry, which IS a JSON object and so is placed at the world origin today — the second silent-input half of that same ticket, so the malformed-entry test uses non-object junk only; undo/transactionality — `B-foliage-mutators-no-transaction` (OPEN) means an undo assertion could only fail today. Ticket item 4 (`landscape.create_grass_type` variety proof) and item 5 (the 473-fixture `ValidParamsNoCrash` sweep) are NOT done here: item 4 is `B-create-grass-type-addzeroed-never-renders`'s own required regression test and that ticket is being fixed concurrently in the same file region, so writing it here would collide and land red; item 5 is a separate mechanical job outside this namespace. **Verification status, stated plainly: NOT COMPILED AND NOT RUN** — the wave's build and suite run are the caller's. What WAS verified: `Content/Python/check_test_ids.py` over the whole plugin returns `SCANNED 4680 id(s), 4680 unique … CLEAN no dot-prefix collisions, no duplicate ids` with all 7 new ids present, so none of them retires an existing leaf and none is retired; every engine symbol used was read from `C:/UE_5.8/Engine/Source/Runtime/Foliage/` rather than recalled (`FFoliageInfo::Instances`, `AInstancedFoliageActor::FindInfo`/`ForEachFoliageInfo`, the `UFoliageType` ctor defaults); and the fixture/teardown shape matches the already-green `TestFoliageAddInstancesPrecedenceHonesty.cpp`. **One unticketed hazard found while writing these, unverified but visible in source:** `foliage.get_instances`' unfiltered branch does `Type->GetPathName()` on the key `ForEachFoliageInfo` hands it (`InstancedFoliage.cpp:3016-3027` passes `Pair.Key` with no guard), and `AInstancedFoliageActor::AddReferencedObjects` (`:5208`) registers that key through the MUTABLE `AddReferencedObject` overload — so a force-deleted foliage type that the IFA still references can leave a null key the unfiltered read would dereference. Not filed, and not reproduced: the existing suite already runs an unfiltered read after force-deleting a referenced type and is green, so either the null does not materialise on this host or something clears the entry. Worth a null guard in the handler regardless.
