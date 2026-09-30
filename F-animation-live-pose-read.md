---
id: F-animation-live-pose-read
title: "No verb reads the evaluated pose or AnimInstance state of a live PIE or editor skeletal mesh component"
status: OPEN
severity: Medium
category: feature
tags: [animation, pie, runtime, bones, anim-instance, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# No live pose / AnimInstance read

`skeleton.get_bone_transform` returns the reference (bind) pose
(`Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp:350`). Reading what a character
is actually doing in PIE is only possible one bone per call via `object.call_function` on
`GetSocketTransform` (bone names are accepted) against a PIE object path (see the
`runtime-uobject-inspection` wiki page), or via `python.execute`. There is no way to read state
machine states, the active montage or curve values in one call.

Competitors ("Live PIE bone reads and AnimBP override" row): ue-mcp `get_live_bone_transforms`
(world auto|pie|editor, actor by path or label, space world|component|local, up to 1000 bones,
passive read of the component's current arrays, `AnimationHandlers_SkeletalLive.cpp:650-811`) and
`set_live_post_process_anim_blueprint` (5.5+, `:514-636`); Monolith `sample_pie_anim_instance`
(world-space bones/sockets, current state per state machine, active montage/section, reflected
variables, `MonolithAnimationRuntimeActions.cpp:352-612`).

## Engine API (UE 5.8)

- `USkinnedMeshComponent::GetComponentSpaceTransforms` `SkinnedMeshComponent.h:1734` (current read buffer), `GetSocketTransform` `:1360`.
- `USkeletalMeshComponent::GetBoneSpaceTransforms` `SkeletalMeshComponent.h:455` (blocks on parallel evaluation, returns a copy).
- `USkeletalMeshComponent::SetOverridePostProcessAnimBP` `:433` — BlueprintCallable, **5.5+ only** (absent in 5.3/5.4, checked).
- `UAnimInstance::GetCurrentStateName` `AnimInstance.h:1269`, `GetStateMachineIndex` `:1202`, `Montage_GetCurrentSection` `:696`, `GetCurrentActiveMontage` `:754`, `GetActiveCurveNames` `:1255`, `GetCurveValue` `:1241`.
- PIE world: `GEditor->PlayWorld`, `GetPIEWorldContext` `EditorEngine.h:2603`.

## Proposed scope

Read-only `animation.get_live_pose`:
- `world` (`editor|pie|auto`, same convention as `actor.list`), `actor` (required, path or label; ambiguous labels refused), `component` (optional), `bones[]` (optional, default all, capped), `space` (`world|component|local`, **required** — no safe default), `include[]` (`stateMachines`, `montage`, `curves`).
- Returns per-bone `{name, index, parentName, transform}`, the resolved world/actor/component, and a freshness marker for the pose buffer (e.g. the component's last tick frame vs `GFrameCounter`) so a stale buffer is distinguishable from a fresh one.
- Leaves every package's dirty flag unchanged.

Post-process AnimBP override: first test whether `object.call_function` can call
`SetOverridePostProcessAnimBP` with a `TSubclassOf` argument (untested). Add a dedicated verb (5.5+,
`SendUnsupportedEngineVersion` below) only if it cannot.

## Acceptance

- In PIE, bone outputs match `GetSocketTransform` for the chosen space within 1e-3.
- `space` omitted is rejected by the required-param gate.
- `world:pie` with no PIE session returns `PIE_NOT_ACTIVE`.
- The response reports how fresh the pose buffer is.
- The `object.call_function` override test result is recorded in this ticket.

**Effort:** S-M.

## History
- `#1-no-live-pose-read` `OPEN` reporter — Filed from the 2026-09-30 animation gap analysis at plugin HEAD `2580e7f4`. Severity Medium: soft blocker (per-bone `object.call_function` workaround exists).
