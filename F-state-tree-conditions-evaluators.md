---
id: F-state-tree-conditions-evaluators
title: "StateTree authoring stops at topology — no evaluators, conditions, transition triggers, or property bindings"
status: DONE
severity: High
category: feature
tags: [state-tree, ai, authoring, imperative-api]
---

# StateTree authoring is topology-only — runtime gating unreachable

The current StateTree surface lives on `ai.*` and covers exactly four
handlers: `ai.create_state_tree`, `ai.add_state_tree_state`,
`ai.add_state_tree_transition`, `ai.configure_state_tree_task`
(AIHandler.cpp lines 1499 / 1587 / 1701 / and the configure handler
further down). That covers the **topology** — asset + states + parent
linkage + a transition between two named states + which task class
runs in a state — and stops well short of authoring a functional
state tree.

A StateTree run-time decision is gated by three first-class concepts
the API never reaches:

1. **Evaluators** (`FStateTreeEvaluatorBase` subclasses stored on
   `UStateTreeEditorData::Evaluators`). Tree-scoped objects that
   produce values every tick for tasks/conditions/transitions to bind
   against. Without them, transitions can only fire on state
   completion or hard-coded events.
2. **Conditions** (`FStateTreeConditionBase`, stored on
   `UStateTreeState::EnterConditions` and per-transition
   `FStateTreeTransition::Conditions`). Without conditions, a state
   cannot be conditionally entered and a transition cannot be gated by
   anything beyond its trigger.
3. **Transition triggers.** `ai.add_state_tree_transition` hard-codes
   `OnStateCompleted` (with no other path through the handler in
   practice — `triggerType` is documented as optional with that
   default but the handler does not branch on it for the major modes).
   `EStateTreeTransitionTrigger` covers `OnTick`, `OnEvent`,
   `OnStateCompleted`, `OnStateSucceeded`, `OnStateFailed`. Without
   `OnTick` and `OnEvent`, event-driven trees (gameplay-event reacts,
   tick-polled checks) cannot be authored.
4. **Property bindings** (`FStateTreePropertyBinding` /
   `UStateTreeEditorData::PropertyBindings`). The wiring layer that
   feeds evaluator outputs into task/condition inputs. Without
   bindings, every node is isolated — evaluators produce nothing
   reachable and tasks read nothing from the tree.

**Use cases blocked:**

1. Authoring a working component-schema StateTree end-to-end without
   dropping into the editor or hand-editing the .uasset. Even a
   trivial "evaluate distance → if < threshold, transition to Attack"
   tree needs an evaluator, a condition, and a binding.
2. Event-driven AI (gameplay-event-triggered transitions) — `OnEvent`
   trigger mode is unreachable.
3. Conditional state entry (`UStateTreeState::EnterConditions`) — any
   non-trivial branching tree.
4. Any binding between tree-scoped data and a task/condition input —
   the entire property-binding layer is invisible.

**Workaround:** None on the imperative surface. There is no StateTree
text-IR analogue of BPIR/AGIR/MGIR. End-to-end authoring requires
opening the asset in the editor.

**Fix:** Native struct-node authoring is the first scope. Add dedicated
`state_tree.*` handlers that resolve evaluator/task/condition
`UScriptStruct` names with `ResolveUScriptStruct`, initialize
`FInstancedStruct` editor nodes, and mutate public StateTree editor data
directly. The implemented surface is `state_tree.add_evaluator`,
`state_tree.add_task`, `state_tree.add_condition`,
`state_tree.set_transition_trigger`, and `state_tree.bind_property`.
Each mutator is undoable through a scoped editor transaction and accepts
`save` defaulting to `false`, leaving the package dirty unless the caller
explicitly requests persistence.
Namespace migration for the older `ai.*` topology handlers remains a
separate follow-up.

**Implementation surface:** `UStateTreeEditorData` is `public` and
already mutated by the existing handlers. `Evaluators` and
`PropertyBindings` arrays are public. `UStateTreeState::Tasks`,
`UStateTreeState::EnterConditions`,
`UStateTreeState::Transitions[i].Conditions`,
`UStateTreeState::Transitions[i].Trigger`, and transition required
event fields are public. Struct resolution uses the existing
`ResolveUScriptStruct` helper. Compile invalidation clears
`StateTree->LastCompiledEditorDataHash`; saving goes through
`McpSafeAssetSave` only when `save=true`.

**Cross-ref:** Same shape of gap as `F-anim-state-machine-internals`
(topology-only imperative surface above an IR escape hatch) — except
StateTree has no IR escape hatch, so the imperative gap is the only
authoring path, raising priority.

## History
- `#1-topology-only-no-runtime-gates` `OPEN` reporter — Current StateTree surface (`ai.create_state_tree`, `ai.add_state_tree_state`, `ai.add_state_tree_transition`, `ai.configure_state_tree_task`, all in `AIHandler.cpp`) authors topology only: asset + states + parent linkage + transition between named states + task class on a state. Evaluators (`FStateTreeEvaluatorBase` / `UStateTreeEditorData::Evaluators`), conditions (`UStateTreeState::EnterConditions`, per-transition `Conditions`), non-`OnStateCompleted` transition triggers (`OnTick`, `OnEvent`, `OnStateSucceeded`, `OnStateFailed`), and property bindings (`FStateTreePropertyBinding` / `UStateTreeEditorData::PropertyBindings`) are all unreachable — no functional StateTree is authorable end-to-end without opening the editor, and unlike anim/material/blueprint there is no text-IR escape hatch. Proposes `state_tree.add_evaluator`, `state_tree.add_condition`, `state_tree.set_transition_trigger`, `state_tree.bind_property` as Phase 1; namespace migration `ai.* → state_tree.*` (mirroring `behavior_tree.*`) as an optional Phase 2 follow-up. Implementation surface (`UStateTreeEditorData::Evaluators` / `PropertyBindings`, `UStateTreeState::EnterConditions` / `Transitions[i].Conditions` / `Transitions[i].Trigger`) is already public and used by the existing handlers.
- `#2-state-tree-authoring-handlers` `IN-REVIEW` developer — Added native `state_tree.add_evaluator`, `state_tree.add_task`, `state_tree.add_condition`, `state_tree.set_transition_trigger`, and `state_tree.bind_property` handlers that resolve `UScriptStruct` nodes, initialize `FInstancedStruct` editor nodes, mutate public StateTree editor data/transition/event/binding fields, and added regression coverage through production handler calls. Counterfactual: reverting production node/binding/trigger mutation leaves `Evaluators`, `Tasks`, transition `Conditions`, editor bindings, or event tag fields empty/default.
- `#3-review-save-transactions` `IN-REVIEW` developer — Added scoped editor transactions before the StateTree mutator `Modify()` calls, changed mutators to default `save=false` with dirty-package behavior unless explicitly requested, and extended regression coverage to assert default calls report `saved=false`.
- `#4-verify-state-tree-authoring` `DONE` tester — Verified: created temp StateTree `/Game/McpVerify/ST_McpVerifyTemp_F_state_tree_conditions_evaluators_6ab21032`, added Root->Attack topology, then live `state_tree.add_evaluator`, `state_tree.add_task`, `state_tree.add_condition`, `state_tree.set_transition_trigger`, and `state_tree.bind_property` returned evaluatorCount/taskCount/conditionCount/bindingCount all `1`, trigger `OnTick`, and default `saved:false`; cleanup `asset.delete path` returned `deletedCount:1` and `existsAfter:false`.
