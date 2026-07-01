---
id: F-eqs-namespace-expansion
title: "Dedicated eqs.* namespace + expanded generator/test enums"
status: DONE
severity: Medium
category: feature
tags: [ai, eqs, env-query, namespace, authoring, missing-namespace]
---

# Dedicated `eqs.*` namespace + expanded generator/test enums

EQS authoring currently lives under `ai.*` and the existing handlers
create transient generator/test objects without appending them to
`UEnvQuery::Options` / `UEnvQueryOption::Tests`. That makes the RPCs
look successful while leaving the query asset effectively unchanged.
The old context and scoring paths mostly echo input instead of
resolving classes or mutating `UEnvQueryTest` fields.

**Proposal:** Add a dedicated `eqs.*` namespace for the first useful
authoring slice:

- `eqs.create(name, path?, save?)` — create a `UEnvQuery` asset.
- `eqs.add_generator(queryPath, generatorType, save?)` — resolve a
  built-in generator name (`ActorsOfClass`, `OnCircle`, `SimpleGrid`,
  `PathingGrid`, `Composite`, `Donut`, `BlueprintBase`) or a generator
  class path, create a `UEnvQueryOption`, store the generator on it,
  append it to `UEnvQuery::Options`, and return `generatorIndex`.
- `eqs.add_test(queryPath, generatorIndex, testType, purpose?, save?)`
  — resolve a built-in test name (`Distance`, `Trace`, `Pathfinding`,
  `PathfindingBatch`, `Dot`, `GameplayTags`, `Overlap`, `Random`,
  `Project`, `Volume`) or a test class path, append it to the selected
  option's `Tests`, and return `testIndex`.
- `eqs.set_context_class(queryPath, generatorIndex, testIndex?,
  propertyName?, contextClass, save?)` — resolve built-in context names
  (`Querier`, `Item`, `NavigationData`, `BlueprintBase`) or context
  class paths and assign the chosen `TSubclassOf<UEnvQueryContext>`
  property on a generator or test.
- `eqs.set_test_filter(queryPath, generatorIndex, testIndex, filter,
  save?)` — set real `UEnvQueryTest` filter fields (`FilterType`,
  `BoolValue`, `FloatValueMin`, `FloatValueMax`) for bool and float
  range/min/max filters.
- `eqs.set_test_scoring(queryPath, generatorIndex, testIndex, scoring?,
  save?)` — set UE 5.6 scoring fields (`ScoringEquation`,
  `ScoringFactor`, `ClampMinType`, `ClampMaxType`, `ScoreClampMin`,
  `ScoreClampMax`, `ReferenceValue`). Reject `curve` input explicitly:
  UE 5.6 `UEnvQueryTest` has no `FRuntimeFloatCurve ScoringCurve`
  field.

**Backward-compat:** Keep `ai.create_eqs_query`,
`ai.add_eqs_generator`, `ai.add_eqs_context`, `ai.add_eqs_test`, and
`ai.configure_test_scoring` registered as deprecated aliases that
delegate to the new production handlers. The new namespace is the
discoverable surface for expanded class resolution and real mutation.

**Use cases blocked today:**

1. Generating real navmesh-projected query points via `PathingGrid`
   plus `Pathfinding` / `PathfindingBatch` / `Project` tests.
2. Tag-gated queries via `GameplayTags` tests.
3. Authoring queries that reference project-defined context classes.
4. Tuning scoring/filter behavior without opening the EQS editor.

**Workaround:** Hand-author the `.uasset` in the EQS editor; use
`asset.dump` to read it back. No imperative authoring path currently
persists generators/tests and configures context, filter, and scoring
fields.

**Cross-ref:** Parallels [`F-chooser-namespace`](F-chooser-namespace.md)
and [`F-gameplay-tags-namespace`](F-gameplay-tags-namespace.md) —
all three carve a dedicated authoring namespace out of an overloaded
legacy surface.

## History
- `#1-narrow-eqs-surface` `OPEN` reporter — Verified the current
  EQS surface in `AIHandler.cpp` (lines 1076–1295): five handlers
  (`ai.create_eqs_query`, `ai.add_eqs_generator`,
  `ai.add_eqs_context`, `ai.add_eqs_test`,
  `ai.configure_test_scoring`); generator enum hard-coded to
  `ActorsOfClass|OnCircle|SimpleGrid`, test enum to
  `Distance|Trace`; `add_eqs_context` echoes `contextType` without
  resolving it to a `UEnvQueryContext` class;
  `configure_test_scoring` only calls `MarkPackageDirty` (no curve
  or equation write); no filter-mode setter; responses omit the
  newly created generator/test index. `docs/board/` grep for
  `eqs|EnvQuery|EQS` returns zero hits — no prior ticket. Proposes
  migrating to dedicated `eqs.*` namespace with six methods
  (`create`, `add_generator`, `add_test`, `set_test_filter`,
  `set_test_scoring_curve`, `set_context_class`), expanded
  generator/test enums covering `PathingGrid`/`Composite`/`Donut`
  generators and `Pathfinding`/`PathfindingBatch`/`Dot`/
  `GameplayTags`/`Overlap`/`Random`/`Project`/`Volume` tests plus
  class-path acceptance for Blueprint subclasses, and backward-compat
  shims under `ai.*` marked deprecated. Mirrors precedent set by
  [`F-chooser-namespace`](F-chooser-namespace.md) and
  [`F-gameplay-tags-namespace`](F-gameplay-tags-namespace.md).
  Flagged that the ticket could split into "enum expansion" vs
  "namespace + typed setters" but kept bundled because typed setters
  are the actual capability blocker — enum expansion alone leaves
  callers unable to configure created tests.
- `#2-eqs-v1-persistence` `IN-REVIEW` developer — Reformulated v1
  around stored `UEnvQuery::Options`, built-in/class-path resolution,
  context assignment, real filter/scoring fields, and deprecated
  `ai.*` shims; implemented `eqs.create`, `eqs.add_generator`,
  `eqs.add_test`, `eqs.set_context_class`, `eqs.set_test_filter`, and
  `eqs.set_test_scoring` in `EQSHandler.*`, delegated legacy EQS
  aliases from `AIHandler.cpp`, rejected unsupported `curve` input
  because UE 5.6 has no `ScoringCurve` field, and added
  `FEqsNamespaceAuthoringPersistsOptionsTest` plus
  `FEqsNamespaceMutatesContextFilterAndScoringTest` to cover the
  counterfactual where reverting persistence/setter code leaves
  options, tests, context class, filter fields, or scoring fields at
  constructor defaults.
- `#3-review-fix-reuse-and-atomic-scoring` `IN-REVIEW` developer —
  Replaced local EQS JSON field readers with shared `JsonUtils`, moved
  loose enum-token normalization to one shared inline helper used by
  EQS and Chooser, delegated EQS create-path cleanup to
  `SanitizeProjectRelativePath`, and validated all
  `eqs.set_test_scoring` clamp enum inputs before mutating any scoring
  fields. Added a regression test for invalid clamp input preserving
  existing scoring values.
- `#4-verify-atomic-scoring` `DONE` tester — Verified: created `/Game/McpTests/EQS/EQS_McpVerify_F_eqs_namespace_expansion`, added `OnCircle` generator and `Distance` test, set baseline scoring, then `eqs.set_test_scoring` with `clampMaxType=not_a_clamp` returned `INVALID_ARGUMENT`; subsequent `property.get` reads on `EnvQueryTest_Distance_0` showed `ScoringEquation=Linear`, `ScoringFactor=(DefaultValue=1.250000)`, `ScoreClampMin=(DefaultValue=2.500000)`, `ScoreClampMax=(DefaultValue=3.500000)`, `ReferenceValue=(DefaultValue=4.500000)`, `bDefineReferenceValue=true`, and both clamp types still `None`.
