---
id: E-ai-bt-authoring-verbs-dead-end
title: "The ai.* BT-authoring verbs (create_behavior_tree / add_composite_node / add_task_node / add_decorator / add_service) are a build-then-throw-away dead end with no upfront steer to behavior_tree.*"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [behavior-tree, ai, discoverability, docs, deprecated-surface, wiki, dead-end]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# Following the `ai.*` BT-authoring verbs builds an unusable asset that must be deleted and rebuilt on `behavior_tree.*`

The `ai` namespace still advertises a full Behavior-Tree-authoring verb set —
`ai.create_behavior_tree`, `ai.add_composite_node`, `ai.add_task_node`,
`ai.add_decorator`, `ai.add_service` — and these surface in the tool list and via
wiki-nav as the obvious first-class way to build a BT from the AI namespace. An
agent that takes the AI namespace at its word and authors a tree through these
verbs walks straight into a dead end:

1. `ai.create_behavior_tree` produces a `UBehaviorTree` with **no `BTGraph`** —
   a later `behavior_tree.decompile` warns "no BTGraph". The asset cannot be
   meaningfully extended.
2. `ai.add_composite_node` (Selector, Sequence) and `ai.add_task_node` (MoveTo,
   Wait) **return success but no node id**, so the nodes are orphaned and
   unaddressable — nothing can be connected to them.
3. `ai.add_decorator` **hard-errors as disabled**:
   `[DEPRECATED_HANDLER] ai.add_decorator created orphaned Behavior Tree
   decorators and is disabled. Use behavior_tree.attach_decorator with assetPath,
   parentNodeId, and decoratorClass.` (This single verb is the only one that
   tells the agent it is on the wrong surface — and only after it has already
   built four orphaned nodes on a graph-less asset.)

The recovery is not a tweak: the agent must `asset.delete` the graph-less
`ai.create_behavior_tree` asset (a naive `behavior_tree.create` at the same path
first collides with `[ASSET_EXISTS]`), then rebuild the *entire* tree on the
working `behavior_tree.*` surface (`create` → `add_node` → `connect_nodes` →
`attach_decorator` / `attach_service`). The `ai.*` half of the work is pure waste.

The disabling of `ai.add_decorator` / `ai.add_service` was the correct call (see
DONE ticket [`F-bt-attach-decorator-service-to-parent`](F-bt-attach-decorator-service-to-parent.md),
which moved the real authoring to `behavior_tree.attach_decorator` /
`attach_service`). The PROCESS gap this ticket files is that the **rest** of the
`ai.*` BT-authoring surface (`create_behavior_tree` / `add_composite_node` /
`add_task_node`) still leads the agent onto the same dead path with **no upfront
steer**: the `ai` overlay (`docs/wiki-src/ai.md`) mentions `behavior_tree` only as
a generic "lower-level graph edits" pointer (line 3) and never warns that the
`ai.*` BT-authoring verbs themselves are orphaning/graph-less and that
`behavior_tree.*` is the *only* surface that produces a runnable tree. So an agent
discovers the dead end one verb at a time, mid-build, instead of being routed to
the right surface before the first call.

This is the same "the namespace advertises a path that dead-ends" friction
already recognized for other surfaces — cf.
[`E-level-structure-wp-wiki-advertises-dead-end`](E-level-structure-wp-wiki-advertises-dead-end.md),
[`E-widget-style-workflow-wiki-advertises-stub`](E-widget-style-workflow-wiki-advertises-stub.md),
[`E-ik-rig-family-wiki-advertises-compiled-out-workflow`](E-ik-rig-family-wiki-advertises-compiled-out-workflow.md)
— applied to the `ai.*` BT-authoring verbs.

## Evidence (this task's call log — `ai.add_decorator` realism task)

`ai.*` BT-authoring path, all wasted before the productive surface even started:

- 5 wiki-nav reads on `ai.*` BT verbs (`create_behavior_tree.md`,
  `add_composite_node.md`, `add_task_node.md`, `add_decorator.md`,
  `add_service.md`) plus `ai`/`get_ai_info` nav.
- `ai.create_behavior_tree` → asset created **with no BTGraph** (decompile later
  warned "no BTGraph").
- `ai.add_composite_node` ×2 (Selector, Sequence) → "no node id returned,
  orphaned"; `ai.add_task_node` ×2 (MoveTo, Wait) → orphaned.
- `ai.add_decorator` ×3 (Blackboard, Cooldown, Loop) → **all `is_error:true`**
  `[DEPRECATED_HANDLER] … is disabled. Use behavior_tree.attach_decorator …`.
- Recovery cost: `behavior_tree.decompile` (confirms no BTGraph) →
  `behavior_tree.create` at the existing path → **`[ASSET_EXISTS]`** →
  `asset.delete` the graph-less asset → `behavior_tree.create` again →
  4× `add_node` → 4× `connect_nodes` → 3× `attach_decorator` →
  `attach_service`. The entire `ai.*` build (1 create + 4 node adds + 3 decorator
  attempts ≈ 8 execute calls, plus ~6 wiki-nav reads) was thrown away.

Friction note (verbatim): *"ai.add_decorator (and the matching
ai.add_composite_node/ai.add_task_node path) are deprecated/orphaning —
add_decorator hard-errors as disabled, and the ai.create_behavior_tree asset had
NO BTGraph (decompile warned 'no BTGraph'), so the ai.* BT-authoring surface is a
dead end. Had to delete and rebuild via the node-id-based behavior_tree.* surface
(create->add_node->connect_nodes->attach_decorator/attach_service)."*

## What it should do (pick one or more)

- **Steer in the docs (cheapest).** In `docs/wiki-src/ai.md`, add an explicit
  note (next to the existing `behavior_tree` cross-ref) that the `ai.*`
  BT-*authoring* verbs (`create_behavior_tree`, `add_composite_node`,
  `add_task_node`, `add_decorator`, `add_service`) are deprecated/orphaning and
  that **all BT graph authoring must go through `behavior_tree.*`**
  (`create` → `add_node` → `connect_nodes` → `attach_decorator` /
  `attach_service`). Mirror the EQS/StateTree pattern already in `ai.md`, which
  redirects new authoring to the dedicated namespace up front.
- **Make `ai.create_behavior_tree` produce a graph-backed asset** (seed the
  `BTGraph` + hidden Root the way `behavior_tree.create` does) so the asset it
  returns is at least extensible on the `behavior_tree.*` surface without a
  delete-and-recreate — and/or have it return the same `rootNodeId` /
  `rootNodeName` that `behavior_tree.create` now returns
  (see [`E-bt-root-entry-node-undiscoverable`](E-bt-root-entry-node-undiscoverable.md)).
- **Disable/redirect the remaining orphaning verbs** the way
  `ai.add_decorator` / `ai.add_service` already are: have
  `ai.add_composite_node` / `ai.add_task_node` either return the node id (so they
  are not silently orphaning) or hard-error with the same
  `[DEPRECATED_HANDLER] … use behavior_tree.add_node …` steer, so the agent is
  routed off the dead surface on the **first** authoring call rather than the
  fifth.

**Workaround:** Ignore the `ai.*` BT-authoring verbs entirely; author every
Behavior Tree on `behavior_tree.*` from the start
(`create` → `add_node` → `connect_nodes` → `attach_decorator` /
`attach_service`), and use `behavior_tree.decompile` (not
`ai.get_ai_info`) to read the structure back.

## History
- `#2-docs-steer-and-disable-orphaning-verbs` `IN-REVIEW` developer — Took the primary docs steer plus the adversarial-recommended consistency redirect; skipped option 2 (graph-backed `ai.create_behavior_tree`) as gold-plating against the active deprecation direction. (1) `docs/wiki-src/ai.md`: added a `## Behavior Tree authoring` section that mirrors the existing EQS/StateTree up-front redirects — it names all five `ai.*` BT-authoring verbs as a deprecated/orphaning dead end (graph-less asset, no node ids, hard-disabled adders) and routes all authoring to `behavior_tree.create -> add_node -> connect_nodes -> attach_decorator/attach_service` + `decompile`; also retoned the prelude (line 3) from "lower-level graph edits" to "all Behavior Tree graph authoring" and updated the `## See also` line. (2) `Source/PinWright/Private/Handlers/AI/AIHandler.cpp`: hard-disabled `ai.add_composite_node` and `ai.add_task_node` with `DEPRECATED_HANDLER` errors steering to `behavior_tree.add_node`/`connect_nodes`, exactly mirroring the already-disabled `ai.add_decorator`/`ai.add_service` — so an agent is routed off the dead surface on the FIRST authoring call instead of the fifth; removed the four now-dead leaf includes (BTComposite_Selector/Sequence, BTTask_MoveTo/Wait). Left `ai.create_behavior_tree`/`ai.configure_bt_node` untouched (the docs steer covers them; disabling create would also block the legitimate assign-an-empty-BT path and was not a listed option). (3) Regression test: repurposed the two stale `MissingRequiredParams` cases in `Source/PinWright/Private/Tests/Gameplay/TestAIHandlers.cpp` into `PinWright.ai.add_composite_node.DisabledOrphaningSurface` and `PinWright.ai.add_task_node.DisabledOrphaningSurface`, which dispatch the real handlers with a plausible payload and assert `bSuccess==false` + `ErrorCode=="DEPRECATED_HANDLER"` + the `behavior_tree.add_node` steer text — these fail if the disablement is reverted (the old body would return NOT_FOUND for the bogus path, not DEPRECATED_HANDLER). Did not compile/run (later phase).
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the `ai.add_decorator` realism task (build a BB + fully-wired patrol BT). The agent followed the `ai.*` BT-authoring verbs and dead-ended: `ai.create_behavior_tree` made a graph-less asset (no BTGraph), `ai.add_composite_node`/`add_task_node` returned no node ids (orphaned ×4), and `ai.add_decorator` ×3 hard-errored `[DEPRECATED_HANDLER] … is disabled`. Recovery required `asset.delete` of the graph-less asset (a `behavior_tree.create` at the same path first hit `[ASSET_EXISTS]`) then a full rebuild on `behavior_tree.*`. ~8 `ai.*` execute calls + ~6 wiki-nav reads thrown away. The judge handled the per-method outcomes; this is the distinct PROCESS angle — the `ai.*` BT-authoring surface is still discoverable and `docs/wiki-src/ai.md` gives no upfront steer to `behavior_tree.*`, so the dead end is discovered one verb at a time mid-build. Same genre as `E-level-structure-wp-wiki-advertises-dead-end` / `E-widget-style-workflow-wiki-advertises-stub`. Proposes a docs steer in `ai.md` (primary), graph-backed `ai.create_behavior_tree`, and/or disabling/redirecting `add_composite_node`/`add_task_node` the way `add_decorator`/`add_service` already are. Cross-refs `F-bt-attach-decorator-service-to-parent` (DONE — disabled `add_decorator`/`add_service`) and `E-bt-root-entry-node-undiscoverable`.
