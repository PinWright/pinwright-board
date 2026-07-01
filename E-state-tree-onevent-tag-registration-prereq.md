---
id: E-state-tree-onevent-tag-registration-prereq
title: "state_tree.set_transition_trigger OnEvent rejects an unregistered gameplay tag [INVALID_PARAMS] — the gameplay_tags.add prerequisite is undocumented on the state_tree overlay"
status: OPEN
severity: Low
category: ergonomic
tags: [state-tree, set_transition_trigger, onevent, gameplay-tags, prerequisite, ordering, discoverability, docs]
encounters: 2
lastSeen: 2026-06-23T10:08:23Z
---

# `state_tree.set_transition_trigger` with an `OnEvent` tag needs the gameplay tag registered first, and nothing on the state_tree overlay says so

`state_tree.set_transition_trigger` accepts an `OnEvent` trigger with a
gameplay-event `tag`. When that tag is **not yet registered** in the project
tag registry, the call hard-fails with:

`[INVALID_PARAMS] Gameplay tag 'AI.Stimulus.Heard' is not registered`

The remedy is to declare the tag first via `gameplay_tags.add`, after which the
identical call succeeds. So the real authoring order for an event-driven
transition is:

`gameplay_tags.add (declare the event tag)` →
`state_tree.set_transition_trigger {triggerType:OnEvent, tag:...}`.

A caller following the natural intent — "wire a Patrol→Investigate transition,
then set its trigger to fire OnEvent on tag `AI.Stimulus.Heard`" — reaches for
`set_transition_trigger` first, eats `[INVALID_PARAMS]`, then has to discover
(not from the docs) that the tag must be registered first. Unlike the GAS
silent-drop case, the error here is **clean and correct** (it names the tag and
says "is not registered") — the only gap is discoverability: the precondition
isn't stated where a caller would look, so it's learned only by eating the
failed call.

The `state_tree` wiki overlay (`docs/wiki-src/state_tree.md`) is a **one-line
descriptive stub** ("Author UStateTree assets — add evaluators, tasks, and
conditions, configure transition triggers, and bind state-tree properties…").
It never mentions that `OnEvent` triggers require a registered gameplay tag,
that `[INVALID_PARAMS] … is not registered` is the failure when the tag is
unknown, or that `gameplay_tags.add` is the way to declare one. So the ordering
dependency is discoverable only via the failed call.

This is the same **overlay-omits-the-precondition** failure mode already tracked
for other namespaces:
[E-navmesh-config-requires-bounds-volume-prereq](E-navmesh-config-requires-bounds-volume-prereq.md)
(a NavMeshBoundsVolume must exist before the nav-config setters resolve),
[E-gas-execution-capture-attribute-set-prereq-undocumented](E-gas-execution-capture-attribute-set-prereq-undocumented.md)
(a hidden `blueprint.compile` step before captures resolve), and
[E-session-wiki-pie-prerequisite-undocumented](E-session-wiki-pie-prerequisite-undocumented.md)
(a live-PIE prerequisite for the local-player/split-screen methods) — here the
missing precondition is "the event tag must be registered via
`gameplay_tags.add`" before an `OnEvent` transition trigger resolves.

This is distinct from
[E-gas-set-effect-tags-drops-unregistered](E-gas-set-effect-tags-drops-unregistered.md):
that ticket is about GAS tag-write verbs *silently dropping* unregistered tags
on a misleading `ok:true` success. Here `set_transition_trigger` already does
the right thing at runtime — it **errors** with a clear `[INVALID_PARAMS]` — so
no behavior change is needed; the fix is purely a wiki-overlay documentation
edit (with optional error-text sharpening to name the remedy).

This is a **process/discoverability** gap, not a wrong result — the method
behaves correctly once the tag exists, the task outcome was clean, and the cost
is one wasted round-trip plus a recovery `gameplay_tags.add` on the first
event-driven transition.

**Evidence (this task — focus `state_tree.add_evaluator`, outcome clean,
27 calls):** the agent built `/Game/AI/StateTrees/BP_GuardBrain`
(Component schema, Root→{Patrol, Investigate}, evaluator + task + condition),
added a Patrol→Investigate transition with `trigger=OnEvent`, then called
`state_tree.set_transition_trigger {Patrol->Investigate, OnEvent,
tag:AI.Stimulus.Heard}` →
`[INVALID_PARAMS] Gameplay tag 'AI.Stimulus.Heard' is not registered`. It
recovered by navigating the `gameplay_tags.add` wiki page, calling
`gameplay_tags.add {tag:AI.Stimulus.Heard}`, then **retrying**
`set_transition_trigger {… save:true}` successfully. Friction note (verbatim):
"state_tree.set_transition_trigger rejected the gameplay tag with
[INVALID_PARAMS] until I registered it via gameplay_tags.add — that
prerequisite is not mentioned on the set_transition_trigger wiki page." Call-log
cost: 1 failed setter call + 1 wiki-nav (`gameplay_tags.add.md`) + 1 recovery
`gameplay_tags.add` + 1 retry — one wasted round-trip plus a register hop, all
to recover an undocumented prerequisite.

**Fix (downstream wiki process; optional error sharpening):** extend the
`docs/wiki-src/state_tree.md` overlay to state that
(a) `state_tree.set_transition_trigger` with `triggerType:OnEvent` (and any
trigger that carries a gameplay-event `tag`) requires the tag to already be
registered in the project tag registry, and fails with
`[INVALID_PARAMS] Gameplay tag '<tag>' is not registered` otherwise; and
(b) the way to declare a tag is `gameplay_tags.add`, so the event-transition
authoring order is `gameplay_tags.add → set_transition_trigger {OnEvent, tag}`.
Optionally sharpen the `[INVALID_PARAMS]` error text to name the remedy
("register it first via gameplay_tags.add"), the way `[NO_GAME_INSTANCE]`
already says "Start Play-In-Editor first." This is a wiki edit (plus an optional
one-line error string), not a behavior change.

## History
- `#2-recurring-onevent-tag-prereq` `OPEN` reporter — Cross-task evidence (re-confirmed, different tags). Another from-scratch StateTree authoring task (focus `state_tree.set_transition_trigger`, namespace `state_tree`, outcome clean, ~30 calls) hit the identical prerequisite. After authoring `/Game/AI/StateTrees/ST_GuardAI` (Component schema, Root→{Patrol, Investigate, Chase}, three transitions), the call `state_tree.set_transition_trigger {Patrol->Investigate, OnEvent, tag:AI.Stimulus.Noise}` failed with `[INVALID_PARAMS] Gameplay tag 'AI.Stimulus.Noise' is not registered`; the agent recovered by calling `gameplay_tags.add {AI.Stimulus.Noise}` and `gameplay_tags.add {AI.Stimulus.Sight}`, then **retried** `set_transition_trigger` for both transitions successfully. Same wasted-round-trip-plus-register-hop cost, same undocumented ordering, now on a second tag pair (`AI.Stimulus.Noise` / `AI.Stimulus.Sight`) — confirming the discoverability gap is recurring, not a one-off. Friction note (verbatim): "set_transition_trigger hard-fails with INVALID_PARAMS if the gameplay event tag isn't pre-registered ('Gameplay tag X is not registered') — neither the wiki page for set_transition_trigger nor add_state_tree_transition mentions you must gameplay_tags.add the tag first; a real user hits a dead-end until they discover the gameplay_tags namespace." Note this evidence newly implicates the `add_state_tree_transition` page too (the friction note names both pages), so the wiki-overlay fix should cover both the trigger-setter and the transition-add docs. No new ticket — appending here per cross-task aggregation.
- `#1-initial-audit` `OPEN` reporter — Process/struggle audit of a State Tree authoring task (focus `state_tree.add_evaluator`, outcome clean, 27 calls). After building `/Game/AI/StateTrees/BP_GuardBrain` and adding a Patrol→Investigate transition, `state_tree.set_transition_trigger {OnEvent, tag:AI.Stimulus.Heard}` returned `[INVALID_PARAMS] Gameplay tag 'AI.Stimulus.Heard' is not registered`; the agent recovered by registering the tag via `gameplay_tags.add` and retrying successfully. Friction note (quoted): "state_tree.set_transition_trigger rejected the gameplay tag with [INVALID_PARAMS] until I registered it via gameplay_tags.add — that prerequisite is not mentioned on the set_transition_trigger wiki page." Call-log cost: 1 failed setter call + 1 wiki-nav + 1 recovery `gameplay_tags.add` + 1 retry. The `state_tree` wiki overlay (`docs/wiki-src/state_tree.md`, a one-line stub) documents no gameplay-tag-registration prerequisite for `OnEvent` triggers and never names `gameplay_tags.add` as the remedy, so the ordering dependency is discoverable only via the failed call. Same overlay-omits-the-precondition failure mode as `E-navmesh-config-requires-bounds-volume-prereq`, `E-gas-execution-capture-attribute-set-prereq-undocumented`, and `E-session-wiki-pie-prerequisite-undocumented`, different precondition (a registered event tag via `gameplay_tags.add`). Distinct from `E-gas-set-effect-tags-drops-unregistered` (which is about *silent* drops on a misleading `ok:true`; here the call correctly errors). Distinct from `F-state-tree-conditions-evaluators` (the DONE feature ticket that built the `state_tree.*` authoring surface — it does not document the tag prerequisite). E-/docs: works once the tag exists; the gap is discoverability + one wasted round-trip. The per-finding judge saw outcome clean and filed nothing (filed_id empty). No existing board ticket on the `set_transition_trigger` OnEvent tag-registration prerequisite (ripgrep across OPEN + closed; qmd unavailable). Proposes documenting the prerequisite + authoring order on `docs/wiki-src/state_tree.md`, with optional `[INVALID_PARAMS]` error sharpening to name the remedy.
