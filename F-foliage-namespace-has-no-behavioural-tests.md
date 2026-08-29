---
id: F-foliage-namespace-has-no-behavioural-tests
title: "No foliage test asserts where a verb put anything: 14 test cases across 6 verbs, of which 6 are ValidParamsNoCrash registration smoke and the rest are input/response honesty — foliage.paint has that smoke test and nothing else, which is the same shape that let the create_grass_type defect ship a persisted broken asset"
status: OPEN
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
