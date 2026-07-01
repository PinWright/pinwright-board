---
id: F-anim-motion-warping-authoring
title: "No authoring path for Motion Warping notify-state configuration on montages"
status: DONE
severity: Medium
category: feature
tags: [animation, motion-warping, notify-state, montage, root-motion-modifier]
---

# No authoring path for Motion Warping notify-state configuration

Motion Warping authoring on a montage is driven by
`UAnimNotifyState_MotionWarping` placed on a montage track. In UE 5.6 the
notify state owns an instanced `RootMotionModifier` object, defaulting to
`URootMotionModifier_SkewWarp`, that carries `WarpTargetName`,
`bWarpTranslation`, `bWarpRotation`, axis flags, rotation type, and
time-clamp settings. Runtime side is `UMotionWarpingComponent`
added per-character — not asset-side and out of scope for editor RPCs.

Pre-fix gateway coverage:

- `animation.authoring.add_notify_state` (in `AnimationAuthoringHandler.cpp`)
  resolves any `AnimNotifyState_*` class by name and spawns an instance
  on `UAnimSequenceBase::Notifies`. So it *can* instantiate
  `AnimNotifyState_MotionWarping`, but it does **not** expose
  configuration: no `WarpTargetName`, no `RootMotionModifier`
  nested fields, no axis/rotation flags. The inner `RootMotionModifier`
  object already exists on the notify-state instance in UE 5.6; script
  authoring needs nested property writes such as
  `RootMotionModifier.WarpTargetName`.
- `add_notify_state` operates on `UAnimSequenceBase`, but montages are
  `UAnimMontage : UAnimSequenceBase` so the cast succeeds — the notify
  lands on `Notifies`, not on a section. Montage section authoring is a
  separate concern: `montage.authoring` handlers exist (sections, slots)
  but none of them place a notify on a section's track at a time range.
- Before this task, there were zero `warp`, `MotionWarping`, or
  `RootMotionModifier` hits in `Docs/rpc-method-reference.generated.md`
  or `Handlers/Animation/`.

**Use cases blocked:**

1. Imperative authoring of a melee-attack montage with a "step to target"
   warp window — must hand-edit the .uasset or open the editor.
2. Round-tripping a montage that uses Motion Warping through
   asset-dump → re-author (the warp config is opaque blob in
   `properties.json`).
3. Generating warp windows from gameplay-design data (attack ranges,
   target offsets) at scale.

**Previous workaround:** Spawn the bare notify-state via
`animation.authoring.add_notify_state` with
`notifyClass: "MotionWarping"`, then leave configuration to a manual
editor pass. No way to script the modifier class or warp target name.

**Fix:** Add
`animation.authoring.set_notify_state_property(assetPath, notifyIndex? or
notifyName?, propertyPath, value, save=true)`. It selects an existing
notify-state event by index or name, resolves `propertyPath` through
`PropertyUtils`, and applies the JSON `value` to the
`UAnimNotifyState*` instance stored in
`FAnimNotifyEvent::NotifyStateClass`. This unblocks Motion Warping via
nested paths such as `RootMotionModifier.WarpTargetName`,
`RootMotionModifier.bWarpTranslation`, and
`RootMotionModifier.RotationType`.

Defer a one-shot `animation.authoring.add_motion_warping_window`
factory until there are 2+ concrete callers. The generic property
setter is enough to configure the default instanced modifier and avoids
per-class Motion Warping logic in the first implementation.

**Implementation notes:**

- `add_notify_state` already resolves notify-state classes by short
  name, so `notifyClass: "MotionWarping"` is the existing call shape.
- The runtime modifier template in 5.6 lives on
  `UAnimNotifyState_MotionWarping::RootMotionModifier` as an instanced
  object. The constructor creates a default
  `URootMotionModifier_SkewWarp`, and
  `UMotionWarpingComponent::AddModifierFromTemplate` duplicates the
  configured object at runtime.
- Per-class config objects (e.g. `URootMotionModifier_SkewWarp` carries
  `WarpTargetName`, `bWarpTranslation`, `bWarpRotation`, `RotationType`)
  are edited through nested property paths on the instanced object.
- The Motion Warping editor module (`MotionWarpingEditor`) is not a
  required dependency — the notify-state class is in the runtime
  `MotionWarping` module which is already commonly enabled.

**Cross-ref:**
[`F-anim-bind-asset-on-player-node`](F-anim-bind-asset-on-player-node.md)
established the precedent of generic-setter-plus-typed-factory for
animation authoring. Same shape applies here.

## History
- `#1-initial-repro` `OPEN` reporter — Zero gateway coverage of Motion Warping authoring. `add_notify_state` in AnimationAuthoringHandler.cpp:814 can spawn `AnimNotifyState_MotionWarping` by short name via the existing `AnimNotifyState_*` prefix resolver, but exposes no way to set the inner `RootMotionModifierConfig` class, `WarpTargetName`, or warp axis/rotation flags. No `warp`/`MotionWarping`/`RootMotionModifier` references anywhere in `Handlers/Animation/` or the generated reference. Proposes (1) a generic `animation.authoring.set_notify_state_property` to unblock configuration (also useful for every other notify-state subclass), and (2) optional one-shot `add_motion_warping_window` factory for ergonomics. Ship (1) first; (2) deferrable until 2+ concrete callers per YAGNI.
- `#2-corrected-root-motion-modifier` `OPEN` developer — Reframed Motion Warping configuration around UE 5.6's instanced `RootMotionModifier` object and nested property paths; deferred modifier-class factory work.
- `#3-set-notify-state-property` `IN-REVIEW` developer — Added `animation.authoring.set_notify_state_property` in `AnimationAuthoringHandler.cpp` to set nested notify-state properties, documented the Motion Warping example, and added `FAuthoringSetNotifyStatePropertySetsNestedObjectPathTest` coverage.
- `#4-verify-motion-warping-property` `DONE` tester — Verified: created temp anim sequence `/Game/App/McpVerify/AS_McpVerifyTemp_FAnimMotionWarpingAuthoring_20260515141317`, added `notifyClass=MotionWarping` notify state `WarpWindow`, ran `animation.authoring.set_notify_state_property` with `propertyPath=RootMotionModifier.WarpTargetName` and `value=AttackTarget`, then `asset.dump` showed `Notifies[0].NotifyStateClass.RootMotionModifier.WarpTargetName` as `AttackTarget` in `properties.json`; temp asset deleted.
