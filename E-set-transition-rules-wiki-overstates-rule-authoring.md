---
id: E-set-transition-rules-wiki-overstates-rule-authoring
title: "animation.authoring wiki claims set_transition_rules gives each transition 'a boolean-evaluated condition' — it only toggles flags, sending callers down a dead-end hunt for the rule body"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [docs, animation, animation-authoring, set-transition-rules, state-machine, transition-rule, wiki]
---

# The wiki over-promises `set_transition_rules`, then there is no API to deliver

`docs/wiki-src/animation.authoring.md` line 74 (the `add_state_machine`
narrative) tells the caller:

> "After this call use `add_state` and `add_transition`, then
> `set_transition_rules` **to give each transition a boolean-evaluated
> condition**."

That sentence is false. `set_transition_rules` writes only four coarse
fields — `crossfadeDuration`, `priorityOrder`, `automaticRule`
(`bAutomaticRuleBasedOnSequencePlayerInState`), and `bidirectional`
(see the generated param spec in
`wiki-generated/animation.authoring.set_transition_rules.md`). It never
authors a `bCanEnterTransition` rule expression. The actual rule body
(a variable getter / comparison wired into
`AnimGraphNode_TransitionResult.bCanEnterTransition`) is **unauthorable
through any current RPC** — `F-anim-state-machine-internals` proposed a
`set_transition_rule` helper for exactly this and it was **deferred**
(only 3 of its 5 RPCs shipped; the rule-body helper is not one of them),
and AGIR explicitly excludes the inline rule body
(`F-agir-cliff-completion` item 10, "out of AGIR's scope").

So the wiki promises a capability that does not exist anywhere in the
tool surface. A caller who reads line 74 at face value (the natural
reading: "to give each transition a boolean-evaluated condition" = "this
is how you set the Speed > threshold rule") will call
`set_transition_rules`, see `success:true`, and reasonably believe the
Speed rule was authored — when in fact the rule body is still empty. The
over-promise is the worst kind of doc gap: it does not just fail to
mention a limitation, it actively asserts a capability the API lacks.

## Why this is PROCESS friction (distinct from the feature/echo tickets)

This is the **docs root cause** of the trial-and-error in the call log,
not the feature gap itself:

- `F-anim-state-machine-internals` (DONE) is the *feature* ticket — it
  argued the rule-body helper *should* exist and then **deferred** it.
  It does not touch the wiki sentence that mis-sells the existing
  `set_transition_rules` as already doing the job.
- `E-set-transition-settings-no-echo` / `E-anim-graph-json-omits-transition-logic-blend`
  are about reading back the *settings* method's advanced fields. Neither
  concerns the *rules* method's doc over-promise.
- The judge's `B-create-anim-blueprint-duplicate-name-crash` is the
  unrelated create-time crash.

The fix here is cheap and lives entirely in the wiki overlay: correct the
sentence so callers do not chase a non-existent capability.

## Friction evidence (this task — animation.authoring, 23 calls, story step 6 = the Speed rules)

The story's step 6 explicitly required Speed-based transition rules
("Idle -> Walk when Speed is greater than a small threshold ..."). The
agent (correctly) added a `Speed` float and set the flag fields, then —
trusting the wiki's "boolean-evaluated condition" claim — went hunting
for where the rule body lives. The call log shows the resulting dead-end
sweep, all triggered by believing the rule should be authorable:

- `anim.decompile_agir` on **ABP_Manny** "to learn rule syntax"
- `blueprint.graph.list_graphs` — "(no rule subgraphs)"
- `blueprint.graph.get_graph_details graphName='Idle to Walk'` →
  `[GRAPH_NOT_FOUND]`
- `blueprint.graph.get_graph_details graphName='Transition'` →
  `[GRAPH_NOT_FOUND]`
- `blueprint.inspect` — "(3 top-level graphs only)"

Friction note verbatim: *"the wiki's add_state_machine/set_transition_rules
pages claim set_transition_rules gives 'a boolean-evaluated condition,'
but reading the plugin source ... revealed it only sets
bAutomaticRuleBasedOnSequencePlayerInState and never authors a Speed
comparison ... the transition bound/rule graphs are not exposed by
list_graphs, blueprint.inspect, or any graphName resolver, so
blueprint.graph.create_node cannot target them."*

Two failed RPC calls + three speculative read calls + a plugin-source
read, all of which the wiki could have pre-empted with one honest
sentence. The agent ended the task with the documented success criterion
unmet (empty rule bodies) — and the *only* signal that this was expected
came from reading C++, not the docs.

## Fix (wiki overlay — downstream wiki process, not this audit)

Edit `docs/wiki-src/animation.authoring.md`:

1. Line 74: replace "`set_transition_rules` to give each transition a
   boolean-evaluated condition" with an accurate description —
   `set_transition_rules` sets coarse transition properties
   (crossfade duration, priority, automatic/sequence-player rule toggle,
   bidirectional). State the limitation explicitly: **it does NOT author
   the transition's rule expression** (the `bCanEnterTransition` body that
   compares a variable like `Speed`). Point at the actual escape hatch —
   raw `blueprint.graph.*` into the transition's rule subgraph — and
   cross-reference `F-anim-state-machine-internals` (the deferred
   `set_transition_rule` helper) so a reader knows the convenience RPC is
   not yet available.
2. Note the discoverability caveat the call log hit: the transition rule
   subgraph is not surfaced by `blueprint.graph.list_graphs` /
   `blueprint.inspect` on a freshly authored state machine the way it is
   on a fully-compiled fixture like ABP_Manny, so callers should not
   expect to find it by name without the helper.

## History
- `#2-wiki-overlay-corrected` `IN-REVIEW` developer — Fixed the wiki over-promise in `Plugins/EditorAutomationRpcGateway/docs/wiki-src/animation.authoring.md`. (1) The `### animation.authoring.add_state_machine` H3 narrative (the former line-74 sentence) no longer says `set_transition_rules` gives "a boolean-evaluated condition" — it now says the RPC sets the four coarse properties (crossfade/priority/sequence-player auto-rule toggle/bidirectional) and explicitly notes it does NOT author the `bCanEnterTransition` rule expression, pointing to the method page. (2) Added a dedicated `### animation.authoring.set_transition_rules` H3 section documenting the limitation honestly: it lists the four coarse fields, states `automaticRule` is the sequence-player auto-transition toggle (not a user condition), states no RPC authors the inline rule body today (`set_transition_rule` singular deferred in `F-anim-state-machine-internals`; AGIR excludes it by design), points at the raw `blueprint.graph.*` rule-subgraph escape hatch, and warns the rule subgraph is not surfaced by `list_graphs`/`blueprint.inspect` and has no `graphName` resolver (the `'Idle to Walk'`/`'Transition'` GRAPH_NOT_FOUND dead-ends from the call log). Verified against `AnimationAuthoringHandler_AnimBlueprint.cpp` set_transition_rules handler (writes only CrossfadeDuration/PriorityOrder/bAutomaticRuleBasedOnSequencePlayerInState/Bidirectional, never bCanEnterTransition). Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` `FWikiHandlerSetTransitionRulesDocumentsNoRuleBodyTest` (`EditorAutomationRpcGateway.infra.wiki_handler.MethodPage.SetTransitionRulesDocumentsNoRuleBody`) renders the `animation.authoring.set_transition_rules` method page via `WikiHandler::RenderPage` and asserts the overlay-exclusive markers `bCanEnterTransition`, `blueprint.graph`, `F-anim-state-machine-internals` are present and the old "boolean-evaluated condition" wording is absent — fails iff the H3 overlay is reverted. Wiki-overlay-only edit; no transport/dispatcher/handler code touched.
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `animation.authoring` `ABP_DinoDragon_Locomotion` locomotion-state-machine task (23 calls; judge filed `B-create-anim-blueprint-duplicate-name-crash` for the create-time crash). PROCESS finding: `docs/wiki-src/animation.authoring.md:74` asserts `set_transition_rules` gives each transition "a boolean-evaluated condition," but the method writes only `crossfadeDuration`/`priorityOrder`/`automaticRule`/`bidirectional` and never authors a `bCanEnterTransition` rule expression — and no RPC does (`set_transition_rule` was deferred in `F-anim-state-machine-internals`; AGIR excludes the inline rule body per `F-agir-cliff-completion` item 10). The over-promise drove the call log's dead-end sweep: `anim.decompile_agir` on ABP_Manny "to learn rule syntax", `blueprint.graph.list_graphs` (no rule subgraphs), two `blueprint.graph.get_graph_details` calls with guessed names `'Idle to Walk'` / `'Transition'` (both `[GRAPH_NOT_FOUND]`), and a `blueprint.inspect` — plus a plugin-source read to discover the docs were wrong. Fix is wiki-overlay-only: correct line 74 to describe the four coarse fields accurately, state that the rule body is not authorable via this RPC, point at the raw `blueprint.graph.*` rule-subgraph path + the deferred `set_transition_rule` helper, and warn that the rule subgraph is not surfaced by `list_graphs`/`blueprint.inspect` on a fresh SM. Deduped: distinct from `F-anim-state-machine-internals` (DONE feature — the deferred capability, not the doc), `E-set-transition-settings-no-echo` / `E-anim-graph-json-omits-transition-logic-blend` (the *settings* method's readback echo), and the judge's create-crash ticket. Severity Medium — the over-promise actively misdirects rather than merely under-documents, and the affected step is a core, common state-machine authoring requirement.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
