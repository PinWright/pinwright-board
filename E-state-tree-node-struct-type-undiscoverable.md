---
id: E-state-tree-node-struct-type-undiscoverable
title: "state_tree.add_evaluator / add_task / add_condition have no discovery path for valid node struct types — the only resolvable concrete evaluator is a test-suite struct (FTestEval_A), learned by grepping engine source"
status: OPEN
severity: Low
category: ergonomic
tags: [state-tree, add_evaluator, add_task, add_condition, struct-discovery, node-types, discoverability, docs]
encounters: 1
lastSeen: 2026-06-24T06:10:13Z
---

# `state_tree.add_evaluator` / `add_task` / `add_condition` give no way to discover which node struct types are valid

`state_tree.add_evaluator`, `state_tree.add_task`, and
`state_tree.add_condition` each take a node **struct type** name — a concrete
`FStateTreeEvaluatorBase` / `FStateTreeTaskBase` / `FStateTreeConditionBase`
subclass resolved via `ResolveUScriptStruct`. There is **no enumeration or
keyword-search surface** for which struct names are valid, and the
`docs/wiki-src/state_tree.md` overlay is a single one-line stub (verbatim:
*"Author UStateTree assets — add evaluators, tasks, and conditions, configure
transition triggers, and bind state-tree properties…"*) that names no struct
types, no discovery move, and no examples. So a caller asked to "add an
evaluator that produces an output value" or "a task that consumes a value" has
no in-band way to learn *which concrete struct to name*, nor which of its
properties are bindable source/target paths.

The discoverability gap is sharper here than for the sibling
`E-configure-slot-behavior-behaviortype-undiscoverable` case, because of how
the StateTree module is populated: the `StateTreeModule` ships **no public
production evaluator struct** — the built-in evaluator/condition structs are
either abstract bases or Blueprint-only, and the **only concrete
`FStateTreeEvaluatorBase` that `ResolveUScriptStruct` resolves is the test-suite
struct `FTestEval_A`** (a unit-test fixture that happens to expose `FloatA` /
`IntA` / `bBoolA` output properties). A naive caller reaching for a "debug/text
evaluator" finds none exists; the working answer is a test-only struct that no
documentation would ever point them to. The agent learned this only by
**grepping the engine source** for a usable evaluator with output properties.
Tasks are slightly better off (`FStateTreeDelayTask` exists with a real
`Duration` input), but again only because the agent already knew the struct name
or read source — the overlay lists neither.

## Why this is a separate ticket

- Not the OnEvent-tag prereq
  ([E-state-tree-onevent-tag-registration-prereq](E-state-tree-onevent-tag-registration-prereq.md)):
  that is about a registration ordering gate on `set_transition_trigger`. This
  is about **which struct type name to pass** to the node-add verbs in the first
  place.
- Not the dump-coverage gap
  ([E-asset-dump-state-tree-topology-invisible](E-asset-dump-state-tree-topology-invisible.md),
  the judge's `filed_id`): that is read-back (the authored binding/topology not
  appearing in `asset.dump`). This is **authoring-time discovery** (what to name
  before the node exists).
- Not the DONE feature ticket
  ([F-state-tree-conditions-evaluators](F-state-tree-conditions-evaluators.md))
  that built `add_evaluator` / `add_task` / `add_condition` / `bind_property`:
  that shipped the *authoring capability* but documented no catalog of valid
  node struct types, so the discovery friction survives the feature.

It is the same **no-enumerated-values / no-discovery-path** failure mode as
[E-configure-slot-behavior-behaviortype-undiscoverable](E-configure-slot-behavior-behaviortype-undiscoverable.md)
(a `behaviorType` with no listed values, learned via an `availableBehaviorTypes`
error echo + a C++ read) and
[E-eqs-context-class-no-project-discovery-path](E-eqs-context-class-no-project-discovery-path.md)
(no documented route from "I need context class X" to listing the candidates).
And it is the StateTree peer of the existing `F-search-api-*` discovery series
([F-search-api-anim-graph-nodes](F-search-api-anim-graph-nodes.md) DONE,
[F-search-api-material-expressions](F-search-api-material-expressions.md),
[F-search-api-native-uclasses](F-search-api-native-uclasses.md) DONE) — each
adds a typed catalog for a node-type family that was otherwise invisible. There
is no analogous catalog for StateTree node structs.

## Evidence (this task — focus `state_tree.bind_property`, namespace `state_tree`, outcome clean, 18 calls)

The task asked the author to "pick concrete built-in StateTree evaluator/task
struct types … that actually exist in this engine so the binding has real source
and target properties to connect." On a from-scratch `/Game/AI/StateTrees/ST_GuardBrain`
(Component schema), the author used `state_tree.add_evaluator FTestEval_A
name=SightEval (outputs FloatA/IntA/bBoolA)` and `state_tree.add_task
FStateTreeDelayTask name=InvestigateTask (input Duration)`, then bound
`SightEval.FloatA -> InvestigateTask.Duration` — all `ok:true`.

Friction note (verbatim): *"The only built-in concrete FStateTreeEvaluatorBase
struct that resolved is the test-suite FTestEval_A (StateTreeModule ships no
public production evaluator struct - all are Blueprint/abstract), so I had to
grep engine source to find a usable evaluator with output props; a true
'debug/text evaluator' built-in does not exist."*

Process cost: the node-add calls themselves were clean, but the **struct-type
selection** required an out-of-band engine-source grep to discover that the only
resolvable concrete evaluator is a test fixture (`FTestEval_A`) and to find its
bindable output property names — none of which the wiki overlay or any RPC
surfaces. Pure process friction surviving a "done" outcome.

## What it should do

Two complementary remedies (either or both):

1. **Docs (E-/docs, minimal):** expand `docs/wiki-src/state_tree.md` from the
   one-line stub into per-method sections for `add_evaluator` / `add_task` /
   `add_condition` that (a) state the type param is a concrete
   `FStateTreeEvaluatorBase` / `FStateTreeTaskBase` / `FStateTreeConditionBase`
   `UScriptStruct` resolved by name or `/Script/Module.FStruct` path; (b) name
   the handful that actually resolve in a stock editor (e.g. `FStateTreeDelayTask`
   for tasks) and **explicitly warn that the only concrete evaluator that
   resolves is the test-suite `FTestEval_A`** — production evaluators are
   Blueprint/abstract, so an evaluator with native output props for binding
   demos means `FTestEval_A` (outputs `FloatA`/`IntA`/`bBoolA`); (c) note that
   bindable source/target property names come from the chosen struct's
   `UPROPERTY` set.
2. **Catalog RPC (F-, follow-up, mirrors the `F-search-api-*` series):** a
   `state_tree.list_node_types` / `state_tree.search_node_types {role:
   evaluator|task|condition}` that walks `TObjectIterator<UScriptStruct>`
   filtered to the three base structs and returns each concrete struct's name +
   module + its bindable input/output property names — so the agent never has to
   grep engine source or guess. This closes the same loop the anim-graph-node /
   material-expression / native-UClass catalogs already close for their
   families.

Either way: no behavior change to the node-add verbs — they already resolve a
valid struct fine; the gap is purely "how does the caller learn a valid struct
name (and its bindable props) without reading engine source."

**Workaround (today):** grep `Engine/Source/.../StateTree*` for concrete
`FStateTreeEvaluatorBase` / `FStateTreeTaskBase` / `FStateTreeConditionBase`
subclasses, or `python.execute` a `TObjectIterator<UScriptStruct>` walk filtered
to those bases; for an evaluator with native output props, the test-suite
`FTestEval_A` (`FloatA`/`IntA`/`bBoolA`) is the only one that resolves.

## History
- `#1-initial-audit` `OPEN` reporter — Process/struggle audit of a from-scratch StateTree authoring task (focus `state_tree.bind_property`, namespace `state_tree`, outcome clean, 18 calls). The task required picking "concrete built-in StateTree evaluator/task struct types … that actually exist in this engine"; the author had to grep engine source because the `StateTreeModule` ships no public production evaluator struct (all Blueprint/abstract) and the only concrete `FStateTreeEvaluatorBase` that `ResolveUScriptStruct` resolves is the test-suite fixture `FTestEval_A` (outputs `FloatA`/`IntA`/`bBoolA`); `FStateTreeDelayTask` (input `Duration`) covered the task side. Friction note (verbatim): "The only built-in concrete FStateTreeEvaluatorBase struct that resolved is the test-suite FTestEval_A (StateTreeModule ships no public production evaluator struct - all are Blueprint/abstract), so I had to grep engine source to find a usable evaluator with output props; a true 'debug/text evaluator' built-in does not exist." Verified the `state_tree` overlay (`docs/wiki-src/state_tree.md`) is a one-line stub naming no struct types, no discovery move, no examples; no RPC enumerates valid `add_evaluator`/`add_task`/`add_condition` struct types (ripgrep across OPEN + closed for `add_evaluator|add_task|add_condition|StateTree*Base|struct.discovery`; qmd unavailable). Distinct PROCESS angle from the two existing state_tree tickets (`E-state-tree-onevent-tag-registration-prereq` = tag-registration ordering on `set_transition_trigger`; `E-asset-dump-state-tree-topology-invisible` = the judge's filed_id, dump read-back) and from the DONE feature `F-state-tree-conditions-evaluators` (shipped the authoring verbs, documented no node-struct catalog). Same no-enumerated-values/no-discovery-path failure mode as `E-configure-slot-behavior-behaviortype-undiscoverable` and `E-eqs-context-class-no-project-discovery-path`; StateTree peer of the `F-search-api-*` catalog series (`F-search-api-anim-graph-nodes` DONE, `F-search-api-material-expressions`, `F-search-api-native-uclasses` DONE). Proposes (a) expanding `docs/wiki-src/state_tree.md` into per-method sections naming the resolvable structs (incl. the `FTestEval_A` caveat) and where bindable property names come from, and optionally (b) a `state_tree.list_node_types`/`search_node_types {role}` catalog RPC mirroring the search-api series.
