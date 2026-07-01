---
id: B-create-blend-space-axis-config-dropped-on-57
title: "animation.create_blend_space silently drops the minX/maxX/gridX it accepts on UE 5.7 — entire axis-config path is #if'd out behind the removed BlendSpaceBase.h"
status: IN-REVIEW
severity: Medium
category: bug
tags: [animation, blend-space, create_blend_space, ue5.7, accepted-param-dropped, dead-branch, set_axis_settings]
---

# `animation.create_blend_space` accepts axis range/grid params but silently drops them on UE 5.7

`animation.create_blend_space` is advertised in the wiki (`animation.md:9`) as a
**one-shot convenience** — *"Reach for these when one call settles the operation."*
The handler reads `minX` / `maxX` / `gridX` from the payload
(`AnimationHandler.cpp:154-156`) and passes them to `ApplyBlendSpaceConfiguration`,
so a caller naturally expects "create a Speed 0..600 1D blend space" to settle
the axis range in that single call.

It does not — on this engine (UE 5.7). `ApplyBlendSpaceConfiguration`'s entire
axis-writing body is guarded by `#if MCP_HAS_BLENDSPACE_BASE`
(`AnimationHandler.cpp:158-184`), and that macro is defined `0` whenever neither
`Animation/BlendSpaceBase.h` nor `BlendSpaceBase.h` is includable
(`AnimationHandler.cpp:38-48`). `UBlendSpaceBase` was deprecated/removed in
favour of `UBlendSpace`, so on UE 5.7 the headers are gone, the macro is `0`, and
the `#else` branch runs: it logs *"BlendSpaceBase headers unavailable; skipping
axis configuration"* and the create response carries
`warning: "BlendSpaceBase headers unavailable; axis configuration skipped."`
(`AnimationHandler.cpp:185-193, 515-522`). The min/max/grid the caller passed are
**accepted and then discarded** — the asset is created with default axis bounds
(0..1), not the requested 0..600.

This is the *only* branch that ever executes on this host: the whole fuzz host is
UE 5.7, so `MCP_HAS_BLENDSPACE_BASE` is always `0` here and **every**
`create_blend_space` call drops its axis params. The "convenience" never settles
the operation, and — contrary to the original report — there is **no working
recovery call** either: `animation.authoring.set_axis_settings` is itself a no-op
stub on UE 5.7 (see Workaround below). Net: a documented one-shot param set is
dead on the current engine, and the second-call "fix-up" the attempt agent ran
also silently fails to apply the range.

This is **distinct** from the two existing blend-space tickets:
- `E-blend-space-grid-divisions-on-axis-settings-undiscoverable` (ergonomic/docs)
  argues the create call has *no* grid-divisions param and you must *discover*
  `set_axis_settings`. But the create call **does** accept `minX`/`maxX`/`gridX`
  — they just silently no-op on 5.7. So the real defect here is not "param is
  missing / undiscoverable" but "param is present, documented, accepted, and
  dropped by a dead `#if` branch on the shipping engine." That is a bug, not a
  discoverability gap.
- `E-get-animation-info-thin-on-blend-space` is read-*back* parity.

The implementation should write the axis range/grid through the **`FProperty`
reflection** path the codebase already uses for this exact protected array,
instead of gating all axis writes on the removed `UBlendSpaceBase`, so
`minX`/`maxX`/`gridX` apply on the create call as advertised.

Note on the engine API: there is **no public `UBlendSpace::SetBlendParameter`**.
`UBlendSpace::GetBlendParameter(int32)` returns a `const FBlendParameter&`
(read-only) and the backing `BlendParameters[3]` is a `protected` UPROPERTY
(`Engine/Classes/Animation/BlendSpace.h:543,907`). The deprecated `#if` branch
only "worked" by `const_cast`-ing the const ref off `UBlendSpaceBase`. The
sibling handlers `animation.authoring.create_blend_space_1d` / `_2d` already
demonstrate the correct, non-deprecated UE-5.7 write:
`UBlendSpace::StaticClass()->FindPropertyByName(TEXT("BlendParameters"))` →
`ContainerPtrToValuePtr<FBlendParameter>(BlendSpace)` → write `.Min/.Max/.GridNum`
(`AnimationAuthoringHandler_BlendSpace.cpp:317-325` for 1D, `424-433` for 2D).

**Workaround:** there is **no working one-call-or-two recovery on UE 5.7.** The
originally-reported workaround (call `animation.authoring.set_axis_settings` after
`create_blend_space`) does **not** apply the range: `set_axis_settings`
(`AnimationAuthoringHandler_BlendSpace.cpp:519-575`) reads `minValue`/`maxValue`/
`gridDivisions` into locals (L559-562) and then writes **nothing** to the blend
parameters — it explicitly comments *"For now, skip direct modification since
BlendParameters is protected"* (L552-557), calls only `PostEditChange()`/
`MarkPackageDirty()`, and returns `{success:true, "Axis settings updated"}`. So it
returns a false success and the asset keeps default 0..1 bounds. The only path
that actually sets axis range at create time today is the **typed**
`animation.authoring.create_blend_space_1d` / `create_blend_space_2d` handlers,
which use the FProperty-reflection write above.

**Fix:** Re-implement `ApplyBlendSpaceConfiguration` to write the protected
`BlendParameters` array via `FProperty` reflection (mirroring
`create_blend_space_1d/_2d`) instead of `#if`-gating it behind the deprecated
`UBlendSpaceBase`, so the `minX`/`maxX`/`gridX` the handler already parses take
effect at creation on UE 5.7. Drop the now-dead
`warning: "BlendSpaceBase headers unavailable; axis configuration skipped."`
branch from the create response. (Separately, `set_axis_settings` should be fixed
to use the same reflection write — out of scope for this ticket but noted here.)

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from an animation locomotion task (build ABP_Locomotion + 1D blend space + state machine + footstep notify on SK_Mannequin_Skeleton; 18 calls, outcome tool_bug). Friction note verbatim: *"animation.create_blend_space warned 'BlendSpaceBase headers unavailable; axis configuration skipped' so the 0-600 range didn't apply at creation and I had to fix it with a follow-up set_axis_settings call."* Call log confirms the create→warn→fix-up shape: `animation.create_blend_space BS_Locomotion_Speed 1D minX0 maxX600` returned ok with `error_text` = *"warning: BlendSpaceBase headers unavailable; axis configuration skipped"*, immediately followed by `animation.authoring.set_axis_settings Speed axis 0-600 grid4` to repair it. Root cause located in source: `ApplyBlendSpaceConfiguration` (`AnimationHandler.cpp:145-200`) parses `minX`/`maxX`/`gridX` (L154-156) but writes them only inside `#if MCP_HAS_BLENDSPACE_BASE` (L158-184); that macro is `0` whenever `BlendSpaceBase.h` is unincludable (L38-48), which is the case on UE 5.7 where `UBlendSpaceBase` is gone — so the `#else` branch (L185-193) skips all axis writes and the create response surfaces the `warning` (L515-522). Deduped: distinct from `E-blend-space-grid-divisions-on-axis-settings-undiscoverable` (claims the param is missing/undiscoverable — it is present and accepted, just dropped) and `E-get-animation-info-thin-on-blend-space` (read-back). Severity Medium: a documented one-shot convenience param is non-functional on the shipping engine and the failure is announced only as a warning string, so the call returns success while silently producing a 0..1 blend space; recoverable via the follow-up `set_axis_settings`.
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORD + fix. Reworded body/Fix: the original Fix named a non-existent `UBlendSpace::SetBlendParameter` (verified absent — `GetBlendParameter(int32)` returns a `const FBlendParameter&` and `BlendParameters[3]` is `protected`, `Engine/Classes/Animation/BlendSpace.h:543,907`), and the original Workaround (recover via `animation.authoring.set_axis_settings`) is itself a no-op on 5.7 (`AnimationAuthoringHandler_BlendSpace.cpp:519-575` reads min/max/grid into locals L559-562, writes nothing, returns false `success:true`). Corrected Fix to the proven FProperty-reflection write the sibling `create_blend_space_1d/_2d` handlers already use (`AnimationAuthoringHandler_BlendSpace.cpp:317-325,424-433`). Fix implemented in `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Animation/AnimationHandler.cpp`: rewrote `ApplyBlendSpaceConfiguration` to write `UBlendSpace::BlendParameters` via `FindPropertyByName(TEXT("BlendParameters"))` → `ContainerPtrToValuePtr<FBlendParameter>` → `.Min/.Max/.GridNum` (returns bool), removed the dead `#if MCP_HAS_BLENDSPACE_BASE` gate and the deprecated `BlendSpaceBase.h` include block, and replaced the create-response's dead `UBlendSpaceBase` cast + "headers unavailable" warning with a single `UBlendSpace` path that reports `axisConfigured`. Regression test added: `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestCreateBlendSpaceAxisConfig.cpp` (`EditorAutomationRpcGateway.animation.CreateBlendSpaceAppliesAxisConfig`) — dispatches the real `animation.create_blend_space` RPC through `FRpcDispatcher::ProcessRequest` requesting a 1D Speed axis 0..600 with 8 grid divisions, then reads the created asset's `GetBlendParameter(0)` back and asserts Min=0/Max=600/GridNum=8; counterfactual: reverting to the `#if`-gated write leaves the default 0..1 / GridNum 4 on 5.7 and the asserts fail. Not compiled here (later phase compiles/runs).
- `#3-additional-set-axis-settings-noop-2d` `IN-REVIEW` reporter — Additional evidence: independent SEED-mode reproduction of the **`animation.authoring.set_axis_settings` silent no-op** on a **2D** blend space (prior evidence was the 1D create path; this directly isolates `set_axis_settings`, the verb the `#2` note flags as the still-unfixed out-of-scope sibling). Task: build BS_DinoLocomotion 2D on SK_DinoDragon_Skeleton, then reconfigure both axes via `set_axis_settings`. REPLAY-CONFIRMED deterministically against the live editor on `BS_DinoLocomotion.BS_DinoLocomotion`: baseline `asset.dump`/blend_space.json showed create defaults (Horizontal `Direction` -180..180 gridNum4; Vertical `Speed` 0..600 gridNum4). Then `animation.authoring.set_axis_settings {axis:"Horizontal", axisName:"TurnRate", minValue:-90, maxValue:90, gridDivisions:8}` → `{"success":true,"message":"Axis settings updated"}`, and `{axis:"Vertical", axisName:"MoveSpeed", minValue:0, maxValue:450, gridDivisions:6}` → `{"success":true,"message":"Axis settings updated"}`. Post-call `asset.dump` blend_space.json was **byte-identical** to baseline — `axisName`/`minValue`/`maxValue`/`gridDivisions` stuck for **neither** axis (still `Direction`/-180..180/grid4 and `Speed`/0..600/grid4). Confirms the false-success contract verbatim: the call returns `success:true, "Axis settings updated"` while writing nothing, so the create→fix-up recovery the `#1` reporter relied on is dead on 5.7 — corroborating `#2`'s note that `set_axis_settings` must get the same FProperty-reflection write (`AnimationAuthoringHandler_BlendSpace.cpp:519-575`). Samples (Dino_Idle@{0,0}, Dino_Walk@{0,600}) survived intact. Not creating a separate ticket: this is the same axis-config write defect this ticket owns; the prior reporter's `set_axis_settings` no-op is documented here (#1/#2) and the `B`-fix's noted-but-out-of-scope follow-up should land it. Outcome tool_bug; culprit `animation.authoring.set_axis_settings`.
- `#4-create-side-range-now-applies-but-axisname-noop` `IN-REVIEW` reporter — Additional evidence (REALISM mode, locomotion AnimBlueprint+BlendSpace scaffold task; outcome of the attempt was done-with-friction). Replay narrows what is still broken on the **running plugin**: the `#2` create-side FProperty-reflection fix is **live and verified** — `animation.create_blend_space {name:BS_HeroLocomotion, dimensions:2, minX:-180, maxX:180, gridX:8, minY:0, maxY:600, gridY:6}` produced an asset whose `get_animation_info` reads back `axes[0] {min:-180, max:180, gridNum:8}` and `axes[1] {min:0, max:600, gridNum:6}` (the exact requested ranges + custom grids, NOT the 0..1/grid4 defaults the ticket body still describes as the current behavior). So the headline "min/max/grid dropped → default 0..1" defect from `#1`/title is **resolved in the binary under test**; the ticket body is now stale on that point. What remains is the **axis-name half**: every axis still reads `displayName:"None"` (verbatim) — `create_blend_space` has no axis-name param, and the recovery `animation.authoring.set_axis_settings {axis:"Horizontal", axisName:"Direction", minValue:-180, maxValue:180, gridDivisions:8}` returned `{"success":true,"message":"Axis settings updated"}` yet the subsequent `get_animation_info` still showed `axes[0].displayName:"None"` — the same false-success no-op `#3` isolated, here narrowed to the `axisName` write specifically (the min/max/grid it also "sets" happen to already match from create, so only the name visibly fails). This is the exact `set_axis_settings` FProperty-reflection follow-up `#2`/`#3` flagged as out-of-scope-but-noted: it must write the protected `BlendParameters[i].DisplayName` (an `FName` — hence the literal `"None"` read-back, i.e. `NAME_None`) the same way the create path now writes `.Min/.Max/.GridNum`. Until then, `create_blend_space`-authored axes are unnamed and there is no working call to name them. Note for the read-back: `get_animation_info` now emits the full `axes[]` array (the `E-get-animation-info-thin-on-blend-space #4` parity fix is also live), which is what makes the `displayName:"None"` visible without an `asset.dump` fallback. Outcome tool_bug; culprit `animation.authoring.set_axis_settings`.
