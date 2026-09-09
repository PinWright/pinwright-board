---
id: F-no-runtime-ai-introspection-bt-node-or-path-status
title: "Nothing reports a *running* AI's state — no active Behavior Tree node, no path-following status, no last move result — and every Python fallback for it is stubbed or absent"
status: IN-REVIEW
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

## History
- `#1-implement-get-runtime-state` `IN-REVIEW` developer — Added the read verb `ai.get_runtime_state` in a NEW file `Source/PinWright/Private/Handlers/AI/AIRuntimeStateHandler.cpp` (+ `AIRuntimeStateHandler.h`) rather than in `AIHandler.cpp`: that file hand-spells ~46 raw error codes and is not in `TestErrorCodeRegistry`'s `PartiallyConvertedHandlerFiles` baseline, so a single `ErrorCodes::ERR_*` reference there would flip it to "adopting" and fail `PinWright.core.error_codes.RegistryAdoptingFilesUseConstantsOnly` on every one of them. The new file uses registry constants only. `AIModule` was already a public dependency in `PinWright.Build.cs`, so no build change was needed. The verb is PIE-only (`McpActorUtils::ResolveQueryWorld("pie", …)`; `NOT_IN_PIE` otherwise), resolves the actor **against the PIE world** through `ActorNameParamUtils::RequireResolvedActor` (so `AMBIGUOUS_ACTOR_NAME` / `ACTOR_NOT_FOUND` come from the shared helper and an editor-world placement copy can never be answered about), and accepts a possessed Pawn or the AIController itself. It returns: `brain` (component class, `isRunning`, `isPaused`, `isBehaviorTree`) with, for a `UBehaviorTreeComponent`, `currentTree`/`rootTree`/`activeInstanceIndex`, `DescribeActiveTrees()`/`DescribeActiveTasks()`, `activeNode` (name, class, executionIndex, treeDepth, staticDescription, `isTask`, and `taskStatus` from `GetTaskStatus` when it is a `UBTTaskNode`) and root-first `activeNodePath` walked through `GetParentNode()`; `pathFollowing` (`GetStatusDesc()`/`GetStatus()`, `hasValidPath`, `hasPartialPath`, `didMoveReachGoal`, `GetCurrentRequestId()`, acceptance radius, path indices, goal actor, and — only behind `HasValidPath()` — `currentTargetLocation` and `pathEndLocation`); and `blackboard` (asset, key count, and every key as `{name,type,value}` via `GetNumKeys`/`GetKeyName`/`GetKeyType`/`DescribeKeyValue`). Only public engine API is used — no private members, no reflection hacks. Two new codes in `Handlers/ErrorCodes.h`: `NO_AI_CONTROLLER` (actor exists in PIE, nothing drives it) and `NO_BRAIN_COMPONENT`. **Deliberate deviation from the ask, surfaced rather than silently chosen:** the ticket says to error "when the actor has no brain", but a plain `AIController` running a bare `MoveTo` with no Behavior Tree has no brain and *is* case A of this ticket ("the pawn is stuck"), so refusing it would withhold the exact `pfStatus` reading the ticket asks for. The verb therefore refuses `NO_BRAIN_COMPONENT` only when the controller has **neither** a `UBrainComponent` **nor** a `UPathFollowingComponent`; a controller with one but not the other is answered, with the missing half reported as `present:false` + `reason`. Honesty contract: each of `brain`/`pathFollowing`/`blackboard` is an object carrying `present:true|false` (+ `reason` when false), and every unreachable scalar is an explicit JSON null with a sibling `<field>Reason` — including `lastMoveResult`, which is **always** null because `UPathFollowingComponent` keeps the last `FPathFollowingResult` in a protected member with no accessor (`DidMoveReachGoal()` is the whole public readback); deriving it from `status` would be the fabricated answer this ticket is about. Documented in `docs/wiki-src/ai.md` — a `## Watching a running AI vs. reading its assets` namespace section (placed before the first `###`, per the overlay rendering rule) and a `### ai.get_runtime_state` method section covering the three error codes, the section/null contract, and how to read an abort/restart loop off successive samples. Tests: `PinWright.ai.get_runtime_state.RefusesOutsidePie` and `PinWright.ai.get_runtime_state.DescribesAbsentStateExplicitly` in `Source/PinWright/Private/Tests/Gameplay/TestAIRuntimeState.cpp`. Counterfactuals — (1) drop the PIE gate and the first test fails: the body falls through to editor-world resolution and reports `ACTOR_NOT_FOUND` instead of `NOT_IN_PIE`; (2) emit a zero vector instead of a null when `HasValidPath()` is false and the second test fails: `currentTargetLocation` becomes a JSON object, so the "written as an explicit null" check (read off `FJsonObject::Values` because `HasField` answers false for nulls) no longer holds. **Untestable in the unit suite, stated rather than faked:** the populated BT branch (`activeNode` / `activeNodePath` / `taskStatus` with a tree actually running). Reaching it needs `StartTree` on a component owned by a possessed AIController inside a ticking PIE world — `InstanceStack` is protected and cannot be seeded from outside — and PIE cannot be started in this suite (starting it under `-unattended` walks dirty transient Blueprints left by sibling tests). A constructed component always answers `TreeHasBeenStarted()==false`, which is the branch the second test does assert.
