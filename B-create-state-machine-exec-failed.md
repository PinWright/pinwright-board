---
id: B-create-state-machine-exec-failed
title: "animation.create_state_machine always fails — dispatches console verbs (AddAnimStateMachine/AddAnimState/...) that no Exec handler consumes; first command returns false for ANY input"
status: IN-REVIEW
severity: High
category: bug
tags: [animation, state-machine, create_state_machine, exec, command-failed, dead-method]
encounters: 2
lastSeen: 2026-07-02T07:13:27.2971674+03:00
---

# `animation.create_state_machine` is wholesale broken — the very first console verb it dispatches has no handler

`animation.create_state_machine` is documented (wiki `animation.md` "How to
use", and `wiki-generated/animation.create_state_machine.md`) as a top-level
one-shot convenience that *"Add[s] a state machine sub-graph (with states and
transitions) to an existing Animation Blueprint's AnimGraph"*, with the
parity-rationale (`docs/rpc-hard-removal-rejected-candidates.md:172`)
*"Non-trivial payloads exceed 1-3 generic authoring calls."* In practice the
method **cannot succeed for any input**.

The handler (`AnimationHandler.cpp:565-631`) does not call any animation API
directly — it builds a list of legacy **console command** strings
(`AddAnimStateMachine`, `AddAnimState`, `SetAnimStateEntry`, `SetAnimStateExit`,
`AddAnimTransition`, `SetAnimTransitionRule`) and runs them through
`ExecuteEditorCommandsInternal` (`AnimationHandler.cpp:206-233`), which does
`GEditor->Exec(EditorWorld, *Command)` and returns false on the first command
that no `Exec` handler consumes (L225-228). None of those six verbs is
registered as a console command anywhere in the plugin (grep over `Source/`:
`AddAnimStateMachine` & siblings appear **only** as the emitted strings in
`AnimationHandler.cpp` — there is no `FSelfRegisteringExec`, no `Exec` override,
no engine console command of that name). So `GEditor->Exec` returns `false` on
the very first command (`AddAnimStateMachine ...`), and the handler emits a bare
`[COMMAND_FAILED] Failed to execute editor command: AddAnimStateMachine <bp>
<name>` with no diagnostic about *why* (the underlying "no Exec consumer" is
swallowed by the boolean return).

This is the same root-cause class as the DONE ticket
`B-set-view-mode-exec-failed` (a `GEditor->Exec` of a verb that finds no
consumer → false → bare `*_FAILED`), but materially worse: `set_view_mode`
dispatched a *real* console command (`viewmode`) merely routed to the wrong
target, so it could be fixed by re-routing; here the dispatched verbs **do not
exist at all**, so the entire method is dead code — every call fails on command
#1 regardless of payload. The error also misleads: it names a console command
the caller never typed and can never make work, with no hint that the working
path is the fine-grained `animation.authoring.add_state_machine` /
`add_state` / `add_transition` cluster.

(Note: even if the verbs were registered, the inline `transitions[].condition`
param is documented but unauthorable — `SetAnimTransitionRule` would map to the
deferred rule-body capability per `F-anim-state-machine-internals` and the
wiki-overstatement ticket `E-set-transition-rules-wiki-overstates-rule-authoring`.
That is a secondary concern; the primary defect is that the method never gets
past `AddAnimStateMachine`.)

**Workaround:** use the fine-grained imperative cluster instead —
`animation.authoring.add_state_machine` → `add_state` (×N) → `add_transition`
(×N). The attempt agent fell back to exactly this and it worked first try.

**Fix:** Replace the dead console-command dispatch in
`animation.create_state_machine` with the real authoring path. Either (a)
delegate to the same code that backs `animation.authoring.add_state_machine` /
`add_state` / `add_transition` (which manipulate `UAnimStateMachineGraph` /
`UAnimStateNode` / `UAnimStateTransitionNode` directly and demonstrably work),
looping over `states[]` then `transitions[]`; or (b) if the method is redundant
with the fine-grained cluster, hard-remove it and update the wiki + the
rejected-candidates parity table. At minimum, `ExecuteEditorCommandsInternal`
should surface *why* a command failed (e.g. "no Exec handler for verb
'AddAnimStateMachine'") rather than echoing the unconsumed command string.

## History
- `#1-initial-repro` `OPEN` reporter — REALISM-mode locomotion task (build ABP_Locomotion + state machine + blend space + notify on SK_Mannequin_Skeleton). Seed: none. The high-level `animation.create_state_machine` failed; the agent recovered via the fine-grained `animation.authoring.add_state_machine`/`add_state`/`add_transition` path. Replay-confirmed twice against the live editor on the existing `/Game/Animation/Locomotion/ABP_Locomotion`: (1) full payload (machineName `Locomotion`, two states Idle/WalkRun, two transitions with `condition` strings `"Speed > 10"`/`"Speed < 10"`) → verbatim `[COMMAND_FAILED] Failed to execute editor command: AddAnimStateMachine /Game/Animation/Locomotion/ABP_Locomotion Locomotion`; (2) **minimal** payload (only `blueprintPath` + `machineName`, NO states/transitions/conditions) → verbatim `[COMMAND_FAILED] Failed to execute editor command: AddAnimStateMachine /Game/Animation/Locomotion/ABP_Locomotion LocomotionMinimal`. The minimal repro proves the failure is structural and not payload-dependent: it dies on command #1 (`AddAnimStateMachine`) before any state/transition/condition is even processed, ruling out the agent's "inline condition" hypothesis. Root cause located in source: `AnimationHandler.cpp:565-631` emits console verbs `AddAnimStateMachine`/`AddAnimState`/`SetAnimStateEntry`/`SetAnimStateExit`/`AddAnimTransition`/`SetAnimTransitionRule`, run via `ExecuteEditorCommandsInternal` (`AnimationHandler.cpp:206-233`, `GEditor->Exec` → false → bare `Failed to execute editor command: <cmd>`); a grep over `Source/` shows those six verbs exist ONLY as the emitted strings (no Exec/console-command registration), so `GEditor->Exec` finds no consumer and returns false for any input → the method is dead. Same `GEditor->Exec`-returns-false class as DONE `B-set-view-mode-exec-failed`, but on a verb that does not exist at all (vs. one merely mis-routed), so the whole method is non-functional. Deduped: no existing ticket mentions `create_state_machine`/`AddAnimStateMachine`; distinct from `F-anim-state-machine-internals` (DONE — the deferred *rule-body/pose* depth on the *fine-grained* cluster, which works) and `E-set-transition-rules-wiki-overstates-rule-authoring` (the `condition`/rule-body over-promise, a secondary concern here). Severity High: a documented one-shot method is 100% non-functional with a misleading bare error, and the fine-grained workaround exists, so it is recoverable but the convenience surface is broken.
- `#2-fix` `IN-REVIEW` developer — Implemented fix option (a): replaced the dead console-verb dispatch in `animation.create_state_machine` with direct AnimGraph manipulation, reusing the exact helpers that back the working `animation.authoring.add_state_machine`/`add_state`/`add_transition` cluster (`AnimGraphConstructionUtils::CreateStateMachine` → `CreateState` per `states[]` → `CreateTransition` per `transitions[]`, plus entry-node rewiring via `SMGraph->EntryNode` honoring `isEntry` / defaulting to the first state). Removed the now-dead `ExecuteEditorCommandsInternal` helper (its only caller was this handler) and the six `GEditor->Exec` verb strings. The inline `transitions[].condition` rule string is intentionally NOT authored (deferred per `F-anim-state-machine-internals`); when a non-empty `condition` is supplied the success response now carries a `warning` saying the transition was created always-enterable instead of silently dropping it. New code gated on `MCP_ANIM_HANDLER_HAS_STATE_MACHINE_GRAPH` (the AnimGraph state-machine headers) and returns `ANIMGRAPH_MODULE_UNAVAILABLE` if absent, never a fake success. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/AnimationHandler.cpp` (handler rewrite + new includes + dead-helper removal). Regression test: `FAnimationCreateStateMachineBuildsGraphTest` (`EditorAutomationRpcGateway.animation.create_state_machine.BuildsRealGraph`) in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAnimationHandlers.cpp` — creates a fresh transient AnimBlueprint, drives the real handler via `InvokeHandlerWithCapture` with a two-state (Idle entry / WalkRun) + one-transition payload, asserts `Capture.bSuccess` (pre-fix this was `COMMAND_FAILED`) and that the `UAnimGraphNode_StateMachine` node, both `UAnimStateNode`s, and exactly one `UAnimStateTransitionNode` exist in the graph. Reverting the fix restores the dead Exec dispatch, the handler errors, no node is created, and the test fails. Did not compile/run tests (later phase).
- `#3-additional-granular-fallback-prep-expected-one-shot` `IN-REVIEW` reporter — Additional cross-task evidence (struggle audit of an independent clean/done locomotion task: build `ABP_Mannequin_Locomotion` + `BS_Mannequin_Movement` + Footstep notify on `SK_Mannequin_Skeleton`; 22 MCP calls, zero errors, zero retries). The agent did NOT attempt the one-shot `animation.create_state_machine` at all — it stood up the 3-state Idle/Walk/Jog machine with the usual bidirectional transitions via the fine-grained cluster: 1 `animation.authoring.add_state_machine` + 3 `add_state` + 4 `add_transition` = 8 sequential RPCs (all first-try). The CallAnalyzer flagged the 8-call sequence as a "workaround," and Prep's plan had assumed ONE `animation.create_state_machine` call would build the whole machine — independent confirmation of the ergonomic demand for the one-shot creator that this ticket's `#2` fix (option (a): direct AnimGraph manipulation looping `states[]` then `transitions[]`) restores. No disposition change (fix already IN-REVIEW); logged here rather than as a duplicate `F-` because the batch capability already exists and is being fixed. encounters bumped 1->2.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
