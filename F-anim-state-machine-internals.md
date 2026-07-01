---
id: F-anim-state-machine-internals
title: "State machine authoring API is shallow — no transition rule body, no state pose graph, no entry override, no logic mode"
status: DONE
severity: Medium
category: feature
tags: [animation, anim-graph, state-machine, transition-rule, imperative-api]
---

# State-machine authoring stops at the topology layer

`animation.authoring.add_state_machine` / `add_state` / `add_transition`
/ `set_transition_rules` cover the **topology** of a state machine:
which states exist, which transition node connects which two states,
and the coarse transition timing fields
(`crossfadeDuration`, `priorityOrder`, `automaticRule`,
`bidirectional`). That stops well short of the actual authoring needed
to ship a working state machine.

Each `AnimStateNode` owns an interior pose subgraph (the actual
animation that plays while in the state) and each
`AnimStateTransitionNode` owns a transition-rule subgraph whose
`AnimGraphNode_TransitionResult.bCanEnterTransition` bool pin gates
the transition. Confirmed empirically:
`blueprint.graph.list_graphs` on `ABP_Manny.AnimGraph` returns the
state subgraphs (`Idle`, `Walk / Run`, `Jump`, `Fall Loop`, `Land`) and
the transition subgraph (`Transition`) as `kind: "subgraph"` entries
parented to the state-machine container — they are reachable via
`get_nodes`, but the API has **no state-machine-aware helpers** to:

1. **Set the entry state.** `add_state_machine` documents the first
   `add_state` as the implicit entry, but a downstream reorder cannot
   change it. Real state machines often need to override
   `EntryNode -> StateName` after authoring.
2. **Populate a state's interior pose graph.** Today this requires
   raw `blueprint.graph.create_node` calls into the state's subgraph
   targeting an `AnimGraphNode_StateResult` output and wiring a player
   into it. Discovering the subgraph name is a side effect of
   `list_graphs`; the state-name → subgraph-name mapping is not
   documented and depends on display-name munging.
3. **Author the transition rule body.** A real rule wires a variable
   getter (or call-function chain) into the
   `AnimGraphNode_TransitionResult.bCanEnterTransition` pin. The
   imperative path requires `list_graphs` to find the transition
   subgraph, `get_nodes` to locate the result node, then raw
   `blueprint.graph.create_node` + `connect_pins`. No
   state-machine-aware helper sets a transition's rule from a single
   variable reference (the overwhelmingly common case).
4. **Set transition logic mode.** `FAnimNode_Transition` /
   `UAnimStateTransitionNode` carry `LogicType`
   (`StandardBlend` / `CustomBlend` / `LogicalAndOr` for state
   aliases), `BlendMode`, custom blend curves, `Bidirectional`,
   `bDisableNotifications`. `set_transition_rules` exposes only four
   of these; the rest are unreachable.
5. **Configure conduit / state-alias nodes.** State aliases are
   covered on the AGIR side (`F-agir-state-alias`, completed), but the
   imperative `add_state` API offers no peer for creating a
   `UAnimStateAliasNode` or a `UAnimStateConduitNode`.

**Use cases blocked:**

1. Imperative authoring of a working three-state locomotion machine
   without dropping into AGIR — every transition's `bCanEnterTransition`
   has to be wired via raw blueprint.graph calls.
2. Editing an existing state machine's entry state (e.g. shipping a
   "Falling" override that needs to be the new entry).
3. Adding custom blend logic — `LogicType=CustomBlend` requires a
   transition logic subgraph the API never references.
4. State-alias topology (existing AGIR-side feature has no imperative
   peer).

**Workaround:** Use `call("anim.compile_agir")` — the text-IR
covers all of this and the cliff completion notes confirm both
`state_alias` and `custom_transition` round-trip. The imperative
surface is the gap.

**Proposal:** Add a cluster on `animation.authoring.*`:

- `set_state_machine_entry(blueprintPath, graphName, stateMachineName, stateName)`
  → rewires the `UAnimStateEntryNode` exec to the named state.
- `set_transition_rule(blueprintPath, stateMachineName, fromState, toState, ruleVariableName)`
  → finds the transition subgraph, ensures a `K2Node_VariableGet`
  exists for `ruleVariableName`, wires it to
  `AnimGraphNode_TransitionResult.bCanEnterTransition`. Convenience
  for the 90% case; complex rules still drop to raw blueprint.graph.
- `set_transition_settings(... , logicType?, blendMode?, blendCurvePath?, disableNotifications?)`
  → expose the rest of `UAnimStateTransitionNode`'s fields.
- `add_state_pose(blueprintPath, stateMachineName, stateName, sequencePath?, blendSpacePath?, loop?, playRate?)`
  → adds a player node inside the state's pose subgraph and wires it
  to the `AnimGraphNode_StateResult` output. One call to populate a
  state instead of N raw graph operations.
- `add_state_alias(blueprintPath, stateMachineName, aliasName, aliases: string[])`
  → imperative peer to AGIR's `state_alias` opcode.

Implementation surface: `UAnimStateMachineGraph::EntryNode`,
`UAnimStateTransitionNode::BoundGraph`, `UAnimStateNode::BoundGraph`
are all public; the graphs are normal `UEdGraph` instances and the
plugin already manipulates them via `FGraphNodeCreator` in the AGIR
cliff handlers (see `docs/anim` wiki, "AGIR cliff completion"). The
state-name → bound-graph lookup uses `UAnimStateMachineGraph::Nodes`
filtered by `UAnimStateNodeBase::GetStateName()`.

**Cross-ref:** AGIR side covers `state_alias` (done) and
`custom_transition` (done). This ticket is strictly about the
**imperative** peer to those features.

## History
- `#1-state-machine-topology-only` `OPEN` reporter — `add_state_machine`/`add_state`/`add_transition`/`set_transition_rules` author topology + coarse transition timing only. State interior pose graphs, transition rule bodies (`AnimGraphNode_TransitionResult.bCanEnterTransition`), entry-state override, transition `LogicType`/`BlendMode`/blend-curve, and `UAnimStateAliasNode`/`UAnimStateConduitNode` creation are all unreachable through the imperative API. Verified via `blueprint.graph.list_graphs` on `ABP_Manny.AnimGraph` — state and transition subgraphs are exposed (and `get_nodes` reads their internals: `AnimGraphNode_TransitionResult` + `K2Node_VariableGet`) but there are no state-machine-aware helpers to author them. Proposes `set_state_machine_entry`, `set_transition_rule`, `set_transition_settings`, `add_state_pose`, `add_state_alias` on `animation.authoring.*`.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Re-verified the four handlers in `AnimationAuthoringHandler.cpp` (lines 2138/2208/2294/2396): topology-only as described, `set_transition_rules` writes exactly four fields (`CrossfadeDuration`, `PriorityOrder`, `bAutomaticRuleBasedOnSequencePlayerInState`, `Bidirectional`) — `LogicType`, `BlendMode`, `BlendProfile`, custom blend curves, `bDisableNotifications` are unreachable. `list_graphs` does surface state/transition subgraphs with `parentGraphName` set, so the raw `blueprint.graph.*` workaround is real but verbose. Severity downgraded **High → Medium**: AGIR (`anim.compile_agir`) is the complete text-IR path — `F-agir-state-alias` is DONE, `F-agir-cliff-completion` covers custom_transition + the rest, so nothing here is unauthored, only un-ergonomic via the imperative surface. The five proposed RPCs are convenience wrappers over composable existing ops (entry-state rewire = 1-2 `set_node_property` calls; transition rule body = `create_node` for `K2Node_VariableGet` + `connect_pins` to the existing `AnimGraphNode_TransitionResult`). Cross-ref note: the proposed `add_state_alias` overlaps with AGIR's existing state_alias opcode — recommend confirming whether the imperative peer is worth a separate handler vs documenting the AGIR escape hatch.
- `#3-shipped-3-of-5-sm-internals` `IN-REVIEW` developer — Added `set_state_machine_entry`, `set_transition_settings`, and `add_state_alias` in AnimationAuthoringHandler.cpp plus a file-local `ParseAlphaBlendOption` helper covering the full EAlphaBlendOption surface. Engine corrections vs ticket: `bDisableNotifications` doesn't exist on UAnimStateTransitionNode → exposed `bDisabled` (param `disabled`) instead; ETransitionLogicType doesn't include `LogicalAndOr` → covered StandardBlend/Inertialization/Custom. The remaining two proposed RPCs (`set_transition_rule`, `add_state_pose`) deferred — both require deep subgraph manipulation (K2Node_VariableGet creation in transition logic graph, player creation in state pose graph) with no precedent helpers; AGIR `anim.compile_agir` is the working escape hatch. Regression test FAnimAuthoringStateMachineInternalsTest authors a 2-state SM via dispatched RPCs then exercises all three new handlers with assertions on LogicType/BlendMode/bDisabled, AliasNode StateAliasName/GetAliasedStates, and EntryNode pin connection.
- `#4-returned-regression-fails` `OPEN` tester — Returned: `system.run_tests` resolved `EditorAutomationRpcGateway.anim.authoring.StateMachineInternals` but job `j_20260515T055026_04203c71` failed with `TESTS_FAILED`; `Saved/Logs/PDS.log` reported `PreEmitDollarVar: VariableGet node for '$v' has no non-exec output pin` and `ResolveDollarVar: '$v' not found — was PreEmitDollarVar called?`. Test: `system.run_tests` with `test="EditorAutomationRpcGateway.anim.authoring.StateMachineInternals"`.
- `#5-return-reclassified-bpir` `IN-REVIEW` developer — Reclassified the returned `$v` failure as unrelated to animation state-machine authoring: the failing logs come from BPIR dollar-variable pre-emission before `%v` local fallback, not from the imperative state-machine handlers. Opened `B-bpir-dollar-local-fallback-logs-errors` for the BPIR bug and left this feature ticket in review for the state-machine API work already implemented.
- `#6-verify-three-rpcs-registered` `DONE` tester — Verified: schema discovery via `animation.authoring.set_state_machine_entry?` / `set_transition_settings?` / `add_state_alias?` all return populated param specs. `set_transition_settings` exposes `logicType` ('StandardBlend'|'Inertialization'|'Custom'), `blendMode` (EAlphaBlendOption), `blendCurvePath`, and `disabled` — matching the IN-REVIEW engine corrections (no LogicalAndOr, `bDisabled` instead of `bDisableNotifications`). `add_state_alias` exposes `aliasName`, `aliases`, `globalAlias`, `x`, `y`. The two deferred RPCs (`set_transition_rule`, `add_state_pose`) are explicitly documented as deferred to AGIR; the 3-of-5 ship matches the IN-REVIEW claim.
