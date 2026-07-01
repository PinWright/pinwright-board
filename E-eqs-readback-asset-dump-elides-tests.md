---
id: E-eqs-readback-asset-dump-elides-tests
title: "asset.dump on a UEnvQuery elides the nested test fields — Options is a plain (non-instanced) object-ref array, so the dump needs an env_query.json sidecar (mirroring state_tree.json) to surface generator/tests/purpose/filter/scoring"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [ai, eqs, env-query, authoring, readback, verification, asset-dump, sidecar, coverage, property-get, wiki, docs]
---

# `asset.dump` on a UEnvQuery emits only the opaque `Options` object-ref — the authored generator/tests are invisible (no `env_query.json` sidecar)

After authoring an EQS query with the `eqs.*` namespace (generator + tests +
context + filter + scoring), the natural next step is "read it back to confirm
each test's purpose/filter/scoring/context persisted." There is **no documented
route for that readback**, and the two obvious candidates pull the agent in
opposite directions:

- **`asset.dump` does not surface the test fields.** Its `properties.json` for a
  `UEnvQuery` serializes only the top-level `Options` array as an **object-ref**
  (e.g. the `EnvQueryOption_0` subobject path) and stops — it does not inline the
  nested `EnvQueryOption_0.Tests[*]` `UEnvQueryTest` fields (`TestPurpose`,
  `FilterType`/`FloatValueMin`/`FloatValueMax`, `ScoringEquation`/`ScoringFactor`,
  context class). So the one verb an agent reaches for to "see the asset" returns
  nothing verifiable about the work it just did.
- **`property.get` on the test subobject paths is the verb that works**, but the
  caller has to *already know* the subobject path shape
  (`<asset>.EnvQueryOption_0:EnvQueryTest_*`, or the `EnvQueryOption_0.Tests`
  array index) to reach it. This is in fact the **canonical, intended** EQS
  readback: the `F-eqs-namespace-expansion` feature's own DONE tester
  (`#4-verify-atomic-scoring`) and the `E-eqs-builtin-token-discovery` developer
  (`#2`) both verified authored tests by `property.get` on `EnvQueryTest_*`
  subobject paths — never via `asset.dump`. The route is right; it is simply
  **never named on the EQS authoring surface**, so a fresh agent discovers the
  `asset.dump` elision the hard way before falling back to per-subobject probes.

Net effect on a clean, fully-successful authoring task: the agent runs
`asset.dump`, finds only the `Options` object-ref, and then issues a cascade of
per-subobject `property.get` reads (one each for `Tests`, `Generator`, and every
test's `TestPurpose` / scoring / filter / context field) to do the deep
verification the user asked for. The work succeeds; the friction is the wasted
`asset.dump` round-trip plus the un-guided pivot to `property.get`.

## Why this is a dump-coverage gap (sidecar), with a docs note

- The root cause is exactly the **non-instanced opaque sub-object** shape the
  plugin already solves with a per-class JSON sidecar. `UEnvQuery::Options` is a
  plain `UPROPERTY() TArray<TObjectPtr<UEnvQueryOption>>` with **no** `Instanced`
  specifier (engine `EnvQuery.h:35-36`); `UEnvQueryOption::Generator`/`Tests` are
  likewise bare `UPROPERTY()` (`EnvQueryOption.h:18-22`). So UHT never stamps
  `CPF_PersistentInstance | CPF_InstancedReference`, the dumper's recursion gate
  (`PropertyExport.cpp:113`) returns false, and the option serializes as a bare
  object-ref path string. The DONE `B-asset-dump-instanced-subobjects-not-recursed`
  recursion fix correctly does **not** fire here — this needs its own per-class
  sidecar, exactly like StateTree's `EditorData` pointer did.
- This is the **same shape and the same fix** as the IN-REVIEW
  `E-asset-dump-state-tree-topology-invisible` (a plain `TObjectPtr` opaque
  pointer → resolved with a `state_tree.json` sidecar via
  `REGISTER_DUMP_JSON_SIDECAR`, mirroring the DONE
  `E-asset-dump-userdefinedstruct-field-list`). EnvQuery needs the analogous
  `env_query.json`. There are already 17 such sidecar builders; a docs-only
  cross-link would document a workaround instead of making `asset.dump` actually
  verify the round-trip the way every other opaque-sub-object asset class now does.
- `property.get` on the test subobjects still works and remains the live
  (non-dump) read path; it becomes a **secondary** note, not the primary fix.
- This is distinct from the two existing EQS discovery tickets, which are about
  *inputs*: `E-eqs-builtin-token-discovery` (built-in token strings) and
  `E-eqs-context-class-no-project-discovery-path` (finding a project context
  class). This ticket is about *verifying the output* of authoring — the readback
  step — which neither covers.

## Evidence (this task's call log + friction note)

`eqs.add_test` "FindBestVantagePoint" authoring task (1 PathingGrid generator, 3
tests: Distance/score, Distance/filter, Trace/filter_and_score; Querier context
on all). Outcome clean — zero failed mutations. The verification phase shows the
elision-then-pivot exactly:

- `asset.dump FindBestVantagePoint` (`ok:true`) — but properties.json carried
  only the `Options` object-ref.
- forced cascade of `property.get` reads to verify: `EnvQueryOption_0.Tests`,
  `EnvQueryOption_0.Generator`, then per-test `TestPurpose` / `ScoringEquation` /
  `ScoringFactor` / `FilterType` / `FloatValueMin` / `FloatValueMax` / `Context`
  (about a dozen `property.get` calls for one "confirm the tests" intent).

Friction note (verbatim):

> "asset.dump's properties.json only serialized the Options object-ref (not
> nested test fields), so deep verification required per-subobject property.get
> reads against the EnvQueryOption_0.<test> object paths."

## What it should do / how to fix

**Sidecar (mirrors the StateTree / UserDefinedStruct fix), via the JSON-sidecar
registry.** Per-class JSON dumps are `REGISTER_DUMP_JSON_SIDECAR` records living
next to each builder, walked by `RunRegisteredJsonSidecars` (the
`E-asset-dump-registry-driven-dispatch` DONE refactor). So the fix is a **new
`Handlers/Asset/EnvQueryDumpBuilder.cpp`** with one
`REGISTER_DUMP_JSON_SIDECAR(TEXT("env_query"), DumpFileNames::EnvQuery, …)` record
keyed on `UEnvQuery::StaticClass()`, mirroring `StateTreeDumpBuilder.cpp` /
`UserDefinedStructDumpBuilder.cpp` — **not** a new inline branch in
`AssetDumpHandler.cpp`. Unlike StateTree, `AIModule` is unconditionally linked
(`Build.cs`), so the EnvQuery headers are always available — no `__has_include`
gate needed.

The builder walks `UEnvQuery::GetOptions()` and serializes the authored topology
into `env_query.json`: the query name + per-option `generatorClass`, and per test
its `testClass`, `purpose` (symbolic `EEnvTestPurpose`), `filter` (symbolic
`EEnvTestFilterType` + `floatMin`/`floatMax` for numeric kinds or `boolValue` for
`Match`), `scoring` (symbolic `EEnvTestScoreEquation` + `factor`), and `comment` —
reading the same `TEnumAsByte<…>`/`FAIDataProvider*Value::DefaultValue` fields the
`eqs.*` authoring handlers write. No `DumpFileNames` baseline hand-edit is needed:
`LoadBaselineDumpFiles` now auto-folds every registered JSON sidecar into the
baseline (only the `DumpFileNames::EnvQuery` constant is added). No aspect-version
bump (brand-new aspect → cache treats a missing entry as must-regenerate).

**Docs note (secondary).** Add a short **"Reading back / verifying a query"** `##`
section to `docs/wiki-src/eqs.md`: point at the new `env_query.json` sidecar as the
one-file verification route, and keep the `property.get`-on-test-subobjects path
(`<EqsAsset>.EnvQueryOption_<g>:EnvQueryTest_<...>`, reading
`TestPurpose`/`FilterType`/`Float*`/`ScoringEquation`/`ScoringFactor`/context) as
the live (non-dump) read.

A further structural alternative (out of scope) is an `eqs.get_query`/`describe`
readback RPC returning the resolved options/tests/context/filter/scoring in one
call — but the sidecar makes `asset.dump` itself verifiable, which is the
established approach.

**Workaround (today):** to verify an authored EQS query before the sidecar lands,
`property.get` the option's test subobjects directly
(`<asset>.EnvQueryOption_0:EnvQueryTest_*`), reading `TestPurpose` /
`FilterType` / `Float*` / `ScoringEquation` / `ScoringFactor` / context.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an `eqs.add_test`
  "FindBestVantagePoint" EQS-authoring task (outcome clean; the only `is_error`
  was a self-correcting `asset.list {classNames}` param miss, logged as
  corroborating evidence on `E-asset-path-vs-assetpath-list-drift #6`, not
  separately fileable). Distinct PROCESS angle no existing ticket covers:
  verifying the authored query has no documented readback route. `asset.dump`'s
  `properties.json` serialized only the top-level `Options` object-ref
  (`EnvQueryOption_0` subobject path) and did **not** inline the nested
  `Tests[*]` `UEnvQueryTest` fields, so deep verification fell back to a cascade
  of per-subobject `property.get` reads against the `EnvQueryOption_0.<test>`
  object paths (TestPurpose / scoring / filter / context — ~a dozen calls for one
  "confirm the tests" intent). The `property.get`-on-test-subobjects route is in
  fact the canonical EQS readback (`F-eqs-namespace-expansion #4` and
  `E-eqs-builtin-token-discovery #2` both verify that way), but it is never named
  on the EQS authoring surface (`docs/wiki-src/eqs.md`, a 2-sentence prelude), so
  the agent discovered the `asset.dump` elision the hard way first. Friction note
  (verbatim): "asset.dump's properties.json only serialized the Options
  object-ref (not nested test fields), so deep verification required
  per-subobject property.get reads against the EnvQueryOption_0.<test> object
  paths." Whether the `asset.dump` elision is a residual gap in
  `B-asset-dump-instanced-subobjects-not-recursed` (DONE; recursed
  `CPF_PersistentInstance|CPF_InstancedReference` props) is a handler question for
  the fix loop — but the docs cross-link to the proven `property.get` route is the
  right primary fix regardless. Proposed: add a "Reading back / verifying a query"
  note to `docs/wiki-src/eqs.md` naming the `property.get` subobject-path readback
  and the `asset.dump` elision, cross-linked from the authoring method sections;
  optionally an `eqs.get_query`/`describe` readback RPC.
- `#2-additional-repro-findshootingpositions` `OPEN` reporter — Second
  independent EQS-authoring task reproducing the identical
  `asset.dump`-elides-then-`property.get`-cascade pattern, confirming it is not
  task-specific. Seed `eqs.create`; authored `/Game/AI/EQS/FindShootingPositions`
  (1 SimpleGrid generator on Querier, 2 tests: Distance/score with InverseLinear
  scoring + float-range filter 200-1500, Trace/filter on a project context),
  outcome clean — every `eqs.*` mutation `ok:true`. The verification step the user
  explicitly asked for ("read it back and tell me how many generator options and
  tests it has, confirm each type/purpose") forced the exact elision-then-pivot:
  `asset.dump FindShootingPositions` (`ok:true`) returned only the `Options`
  object-ref, so deep verification fell back to ~11 per-subobject `property.get`
  reads (`EnvQueryOption_0.Generator`, `EnvQueryOption_0.Tests`, then per-test
  `TestPurpose` / `ScoringEquation` / `ScoringFactor.DefaultValue` / `FilterType` /
  `FloatValueMin` / `FloatValueMax` / `Context`). Replay-confirmed the dumper
  elision is current via `mcp__editor-automation__call`: `asset.dump` on the
  engine example `EQS_StateTreeAI_PointsAround` wrote a `properties.json` whose
  `Options` is a bare `TArray` of one object-ref string
  (`...EQS_StateTreeAI_PointsAround:EnvQueryOption_0`) — `Generator`/`Tests` not
  inlined — so the gap survives `B-asset-dump-instanced-subobjects-not-recursed`
  (DONE). Same docs-first fix (name the `property.get` subobject-path readback +
  the `asset.dump` elision in `docs/wiki-src/eqs.md`, cross-linked from the
  authoring verbs). Friction note (verbatim): "asset.dump on an EnvQuery only
  serializes the top-level Options array as a subobject reference; it does NOT
  recurse into the EnvQueryOption to show generator class, test types, purposes,
  scoring, or filter bounds, so dump alone cannot verify the round-trip — I had to
  fall back to property.get against the EnvQueryOption_0/test subobject paths …
  There is no eqs read/inspect verb."
- `#3-additional-repro-findcovernearplayer` `OPEN` reporter — Third independent
  EQS-authoring task reproducing the identical `asset.dump`-elides-then-
  `property.get`-cascade pattern (seed `eqs.set_context_class`). Authored
  `/Game/AI/EQS/EQS_FindCoverNearPlayer` (1 ActorsOfClass generator, 2 tests:
  Distance/score with InverseLinear scoring, Trace/filter bool=true; a created
  `EQS_Context_Player_C` `EnvQueryContext_BlueprintBase` context assigned to both
  the generator `SearchCenter` and the Trace `Context`), outcome clean — every
  `eqs.*` mutation `ok:true`. The user explicitly asked to "dump the asset and
  confirm the context-class assignments and tests are persisted," which forced the
  exact elision-then-pivot: `asset.dump EQS_FindCoverNearPlayer` (`ok:true`) wrote a
  `properties.json` whose only entries were `QueryName` and a bare `Options` TArray
  of one object-ref string, so deep verification fell back to ~9 per-subobject
  `property.get` reads (`EnvQueryOption_0.Generator`, `EnvQueryOption_0.Tests`, then
  per-test `Context` / `ScoringEquation` / `TestPurpose` / `BoolValue`).
  Replay-confirmed the dumper elision is current via `mcp__editor-automation__call`:
  re-running `asset.dump /Game/AI/EQS/EQS_FindCoverNearPlayer` wrote
  `properties.json` =
  `{"Options":{"is_overridden_locally":true,"type":"TArray","value":["/Game/AI/EQS/EQS_FindCoverNearPlayer.EQS_FindCoverNearPlayer:EnvQueryOption_0"]},"QueryName":{...}}`
  — `Generator`/`Tests`/context/scoring/filter not inlined; `meta.json` shows
  `className: EnvQuery`, `propertiesStatus: {status:"n/a", reason:"non_blueprint_asset"}`
  (the EnvQuery falls into the generic-UObject dump bucket, never recursing the
  instanced `Options`/`Tests` subobjects). So the gap survives
  `B-asset-dump-instanced-subobjects-not-recursed` (DONE) and is task-independent
  across three distinct queries now. Same docs-first fix (name the `property.get`
  subobject-path readback + the `asset.dump` elision in `docs/wiki-src/eqs.md`,
  cross-linked from the authoring verbs; optional `eqs.get_query`/`describe`
  readback RPC). Friction note (verbatim): "the success-check's asset.dump is
  INSUFFICIENT for EnvQuery verification — properties.json only listed top-level
  Options/QueryName and serialized the generator/tests as opaque subobject reference
  paths, never expanding the context-class/scoring/filter fields, so I had to fall
  back to property.get on the nested option/generator/test subobjects to actually
  confirm the round-trip (a dump-coverage gap for EnvQuery)."
- `#4-reword+sidecar` `IN-REVIEW` developer — Reworded from docs-only to a
  dump-coverage **sidecar** fix, matching the established `REGISTER_DUMP_JSON_SIDECAR`
  pattern for non-instanced opaque sub-objects (the same shape and fix as the
  IN-REVIEW `E-asset-dump-state-tree-topology-invisible` `state_tree.json` and the
  DONE `E-asset-dump-userdefinedstruct-field-list` `user_defined_struct.json`).
  Confirmed the root cause at engine source: `UEnvQuery::Options`
  (`EnvQuery.h:35-36`) and `UEnvQueryOption::Generator`/`Tests`
  (`EnvQueryOption.h:18-22`) are bare `UPROPERTY()` with no `Instanced` specifier,
  so the dumper's `CPF_PersistentInstance|CPF_InstancedReference` recursion gate
  (`PropertyExport.cpp:113`) never fires — `B-asset-dump-instanced-subobjects-not-recursed`
  (DONE) is correctly inert here; a per-class sidecar is required.
  **Changed:** added `Handlers/Asset/EnvQueryDumpBuilder.{h,cpp}` — a
  `REGISTER_DUMP_JSON_SIDECAR(TEXT("env_query"), DumpFileNames::EnvQuery, …)` keyed
  on `UEnvQuery::StaticClass()` that walks `UEnvQuery::GetOptions()` and serializes
  `queryName` + per-option `generatorClass` and per-test `testClass` / `testOrder` /
  `purpose` / `comment` / `filter` (symbolic `EEnvTestFilterType` + `floatMin`/`floatMax`
  for numeric kinds or `boolValue` for `Match`) / `scoring` (symbolic
  `EEnvTestScoreEquation` + `factor`), reading the same `TEnumAsByte<…>` /
  `FAIDataProvider*Value::DefaultValue` fields the `eqs.*` authoring handlers write.
  No `__has_include` gate (AIModule is unconditionally linked in `Build.cs`). Added
  `DumpFileNames::EnvQuery = "env_query.json"` (`AssetDumpHandler.h`); no
  `FixedCanonical[]` hand-edit needed — `LoadBaselineDumpFiles` now auto-folds every
  registered JSON sidecar into the diff baseline. No aspect-version bump (brand-new
  aspect). Also added the secondary `## Reading back / verifying a query` note to
  `docs/wiki-src/eqs.md` (env_query.json sidecar primary; `property.get` on the
  `EnvQueryOption_<g>:EnvQueryTest_*` subobjects as the live read).
  **Test:** `Tests/Assets/TestEnvQueryDumpBuilder.cpp` (two
  `IMPLEMENT_SIMPLE_AUTOMATION_TEST`s mirroring `TestStateTreeDumpBuilder.cpp`)
  authors a transient `UEnvQuery` with one option (a SimpleGrid generator + a
  Distance test: FilterAndScore, Range filter 100..900, InverseLinear scoring
  factor 2.0; generator/test classes resolved by reflection) and asserts (a)
  `BuildEnvQueryJson` surfaces `options[0].generatorClass`, the test's
  `testClass`/`purpose`/`comment`, `filter.type`+`floatMin`/`floatMax`, and
  `scoring.equation`+`factor`; and (b) `AssetDumpHandler::DumpSingleAsset` writes a
  parseable `env_query.json` with a non-empty `options` array. Both would fail if
  the sidecar were reverted (today the dump surfaces only the opaque `Options`
  object-ref in `properties.json`). Not compiled/tested here (later phase).
- `#5-additional-env-query-sidecar-omits-context` `OPEN` reporter — Fourth
  independent EQS-authoring task (seed `ai.add_eqs_generator`; authored
  `/Game/AI/EQS/FindCoverPoints`: SimpleGrid generator on Querier + ActorsOfClass
  generator, 2 tests on the grid — Distance/filter_and_score with float-range
  filter 200-1500 + InverseLinear scoring, Dot/score), outcome clean — every
  mutation `ok:true`, all fields round-trip-verified. The user's step 10
  ("read the query back and tell me the generator options and the tests
  configured on each") forced the readback. **Materially new since `#1`–`#3`:
  `asset.dump` on a `UEnvQuery` now also writes an `env_query.json` sidecar that
  DOES inline most of what those entries said was elided** — replay-confirmed via
  `mcp__editor-automation__call`: `asset.dump /Game/AI/EQS/FindCoverPoints_Replay`
  wrote `env_query.json` containing both generator classes
  (`EnvQueryGenerator_SimpleGrid`, `EnvQueryGenerator_ActorsOfClass`) and, per
  option, the full `tests[]` with `testClass`, `purpose`
  (`FilterAndScore`/`Score`), `filter` (`{type:"Range", floatMin:200,
  floatMax:1500}`), and `scoring` (`{equation:"InverseLinear", factor:1}` /
  `{equation:"Linear", factor:1}`). So the original "dump shows nothing but the
  Options object-ref" framing is now overtaken by the sidecar for generator/test
  fields (the bare `Options` object-ref persists only in `properties.json`, which
  is no longer the right file to look at). **The residual readback gap is the
  context class:** `eqs.set_context_class {genIdx:0, contextClass:querier}`
  succeeded and persisted (`property.get` on
  `…FindCoverPoints_Replay:EnvQueryOption_0.EnvQueryGenerator_SimpleGrid_0`
  `GenerateAround` returns `/Script/AIModule.EnvQueryContext_Querier`), but the
  `env_query.json` sidecar emits NO context field for either the generator
  (`GenerateAround`) or any test (`Context`) — so the one authoring field this
  task's step 4 set is the only field the structured dump readback still cannot
  confirm, leaving the same `property.get`-on-subobject pivot for the context
  alone. Two doc consequences: (a) `docs/wiki-src/asset.dump.md` does not list the
  `env_query.json` EnvQuery sidecar in its "Files by asset type" / schema summary
  at all (it documents material_instance/data_table/texture/etc. sidecars but not
  EnvQuery), so an agent has no signal the rich EQS dump exists; (b) the proposed
  `docs/wiki-src/eqs.md` readback note should now say: for generators/tests/
  purpose/filter/scoring read `env_query.json` from `asset.dump`, but for the
  context class fall back to `property.get` on the generator/test subobject's
  `GenerateAround`/`Context` property (or, structurally, an `eqs.get_query` verb
  or adding context to the sidecar would close it fully). Friction observed: the
  attempt agent reported "none" only because it cross-checked `env_query.json`;
  the context assignment was never surfaced by any structured readback and would
  have been silently unconfirmed on a generator-context-centric task.
