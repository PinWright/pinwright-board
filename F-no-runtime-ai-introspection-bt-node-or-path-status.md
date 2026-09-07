---
id: F-no-runtime-ai-introspection-bt-node-or-path-status
title: "Nothing reports a *running* AI's state — no active Behavior Tree node, no path-following status, no last move result — and every Python fallback for it is stubbed or absent"
status: OPEN
severity: High
category: feature
tags: [ai, behavior-tree, pathfollowing, runtime, pie, introspection, debugging, moveto]
encounters: 1
lastSeen: 2026-09-07T08:04:00Z
---

# A running AI cannot be observed, only authored

The `ai` and `behavior_tree` namespaces can build a Behavior Tree, a Blackboard and an EQS query,
and `ai.get_ai_info` / `behavior_tree.decompile` read those **assets** back. Nothing reads the
**controller that is executing them**. When an AI does the wrong thing in PIE there is no supported
way to ask which node it is in.

The three facts a stuck AI turns on, and what is available for each today:

| Question | Route tried | Result |
| --- | --- | --- |
| Which BT node is active right now? | `BrainComponent` via Python | `BehaviorTreeComponent` exposes `is_active`, `set_active`, `toggle_active` and nothing else — no active node, no debug string |
| Is path following moving, idle, or did it fail? | `PathFollowingComponent.get_status()` / `has_valid_path()` | `AttributeError` — neither is exposed |
| What did the last `MoveTo` return? | `AIController.get_move_status()` | returns `-1` |

`ai.get_ai_info` does not help: on `controllerPath` it returns `{"controllerClass": "<Controller>_C"}`
and, by its own wiki note, "reads the generated class name and **nothing else**". Its
`behaviorTreePath` branch reports `behaviorTreeName` + `hasRootNode` — the asset, not the instance.

## Where it bit (this checkout, 2026-09-07)

Three enemies pick a cover point and never arrive: over two runs (528 and 273 samples, 90 s and
60 s, 2 Hz) cover was assigned in **100% of samples** and reached in **23% / 25% of cover episodes**,
with the distance-to-cover sitting at a near-constant 283 uu — the EQS grid pitch — while the pawns
moved at a steady 190 uu/s.

Two candidate causes, and they call for opposite fixes:

- **A** — the pawn is stuck: path following has a valid path, `MoveTo` is running, the capsule
  cannot translate.
- **B** — the branch is being aborted and restarted by a decorator observer faster than `MoveTo`
  can finish, so each restart re-runs the EQS from the pawn's new position and the target recedes.

One reading of `pfStatus` and the active task name separates them in a single sample. Neither is
readable, so the stream is reduced to inferring intent from position deltas across RPC calls, which
is how a whole verification slot gets spent on a question a debugger answers instantly.

Worth noting what a naive Python fallback gets *wrong* here: `GetCurrentAcceleration()` reads
`(0,0,0)` on every sample even while the pawn is plainly moving, because AI path following drives a
character through `RequestDirectMove` (which sets `RequestedVelocity`) rather than through
acceleration input. A caller who has no supported status field and reaches for acceleration instead
will conclude "the mover is not even trying" and be wrong. A stubbed surface does not just withhold
the answer; it steers toward a false one.

## Ask

A read verb — `ai.get_runtime_state {actorName | actorPath}` — returning, for a possessed AI:

- **Behavior tree**: active node name and class, the path from root, and the node's execution
  status; ideally the last few node transitions, which is what tells an abort/restart loop apart
  from a long-running task.
- **Path following**: status (`Idle` / `Waiting` / `Paused` / `Moving`), `hasValidPath`,
  current goal location, `didMoveReachGoal`, and the last `EPathFollowingResult`.
- **Blackboard**: the live key/value set (this is the one part that *is* reachable today, via the
  component, so it is the cheapest half to include and makes the response self-contained).

Read-only, PIE-only, one actor per call — no per-actor sweep of the level, which is its own hazard.

## Workaround

None that answers the question. Position and velocity can be sampled across separate
`python.execute` calls and differentiated offline; that shows *that* an AI is not arriving and never
*why*. Note also that `time.sleep()` inside `python.execute` blocks the game thread, so successive
samples inside one call return identical values — sampling has to be one RPC per sample, which makes
a fine-grained trace expensive.

## Notes

- Related but distinct from `E-ai-bt-authoring-verbs-dead-end` (deprecated `ai.*` authoring verbs)
  and `F-bt-composite-child-order-not-authorable` (BT structure not editable after creation). Those
  are about writing a tree; this is about watching one run.
- The engine already computes all of it for the Gameplay Debugger and the BT editor's debugger, so
  the data exists at runtime — it is the exposure that is missing.
