---
id: E-set-ai-movement-currentvalues-partial
title: "ai.set_ai_movement currentValues readback omits brakingDeceleration and avoidanceWeight — echoes 5 of the 7 just-set params, dropping exactly the two without a sibling reader"
status: OPEN
severity: Low
category: ergonomic
tags: [ai, set-ai-movement, movement, currentValues, readback, round-trip, braking-deceleration, avoidance-weight]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
---

# `ai.set_ai_movement` `currentValues` is an asymmetric confirmation block — it echoes 5 of the 7 settable params and silently drops `brakingDeceleration` + `avoidanceWeight`

`ai.set_ai_movement` writes up to 9 movement fields on a blueprint's
`UCharacterMovementComponent` and returns a `currentValues` object that presents
as the per-call confirmation readback. But `currentValues` only re-reads **5**
fields off the just-mutated component:

```cpp
// AIHandler.cpp:2765-2771
CurrentValues->SetNumberField(TEXT("maxWalkSpeed"),    MovementComp->MaxWalkSpeed);
CurrentValues->SetNumberField(TEXT("maxAcceleration"), MovementComp->MaxAcceleration);
CurrentValues->SetNumberField(TEXT("rotationRateYaw"), MovementComp->RotationRate.Yaw);
CurrentValues->SetBoolField(TEXT("orientRotationToMovement"), MovementComp->bOrientRotationToMovement);
CurrentValues->SetBoolField(TEXT("useRVOAvoidance"), MovementComp->bUseRVOAvoidance);
```

So when a caller sets the full guard-tuning param block, the response confirms
`maxAcceleration`/`rotationRateYaw` (and the two bools) but **silently omits
`brakingDeceleration` and `avoidanceWeight`** — two of the seven params the same
method just wrote and listed in `propertiesSet`. The omission is the misleading
part: `currentValues` is not "all settable params" and is not "only the scalar
params" — it is an arbitrary 5-field subset that happens to drop exactly the two
fields (`BrakingDecelerationWalking`, `AvoidanceWeight`) that have **no sibling
reader** anywhere in the `ai` namespace (`ai.get_ai_info` has no movement branch
at all — its `controllerPath` route returns only `controllerClass`; see its wiki
Notes). The caller is left with `propertiesSet` telling them the field *was*
written, but with no way to confirm the *value* from this method's own response —
even though three other settable values right next to them ARE echoed.

This is not a tool bug: every param the caller passed is applied and persists
(replay-confirmed below via `property.get` on the CDO). The setter is correct;
the friction is purely that its `currentValues` block looks like a full
confirmation readback but is a partial, asymmetric one, so a caller who reads it
to confirm the round-trip is misled about `brakingDeceleration`/`avoidanceWeight`.

Same write-only-value / partial-reader class as the `character` namespace's
`get_character_info` omissions (`F-character-info-no-friction-braking`,
`E-character-readback-fallback-undocumented`), but a distinct namespace + method:
here the partial reader is the **setter's own inline `currentValues` echo**, not
a separate getter — different method, different fields, filed separately.

## Repro (replay-confirmed)
1. `blueprint.create {name:"BP_GuardEnemy", savePath:"/Game/AI/Enemies", parentClass:"Character", waitForCompletion:true}` -> ok.
2. `ai.set_ai_movement {blueprintPath:"/Game/AI/Enemies/BP_GuardEnemy", maxWalkSpeed:240, maxAcceleration:1024, brakingDeceleration:1800, rotationRate:360, orientRotationToMovement:true, useRVOAvoidance:true, avoidanceWeight:0.6}` ->
   ```json
   {"blueprintPath":"/Game/AI/Enemies/BP_GuardEnemy",
    "propertiesSet":["MaxWalkSpeed","MaxAcceleration","BrakingDecelerationWalking","RotationRate","bOrientRotationToMovement","bUseRVOAvoidance","AvoidanceWeight"],
    "propertyCount":7,
    "currentValues":{"maxWalkSpeed":240,"maxAcceleration":1024,"rotationRateYaw":360,"orientRotationToMovement":true,"useRVOAvoidance":true}}
   ```
   `propertiesSet` lists all 7 (incl. `BrakingDecelerationWalking`, `AvoidanceWeight`) but `currentValues` echoes only 5 — **no `brakingDeceleration`, no `avoidanceWeight`**.
3. Both omitted values DID persist (so it is a readback gap, not a lost write) — `property.get` on the CDO confirms:
   - `property.get {objectPath:"/Game/AI/Enemies/BP_GuardEnemy.Default__BP_GuardEnemy_C", propertyName:"CharacterMovement.BrakingDecelerationWalking"}` -> `value:1800`.
   - `property.get {objectPath:"/Game/AI/Enemies/BP_GuardEnemy.Default__BP_GuardEnemy_C", propertyName:"CharacterMovement.AvoidanceWeight"}` -> `value:0.6000000238418579`.

## Fix (additive; low-risk)
Mirror the two missing fields in the `currentValues` block (`AIHandler.cpp:2765-2771`),
on the same already-dereferenced `MovementComp`, so the echo confirms every
scalar param the method can set:

```cpp
CurrentValues->SetNumberField(TEXT("brakingDeceleration"), MovementComp->BrakingDecelerationWalking);
CurrentValues->SetNumberField(TEXT("avoidanceWeight"),     MovementComp->AvoidanceWeight);
```

(Field names mirror the input params, like the existing `maxWalkSpeed`/`maxAcceleration`
entries.) Purely additive — existing `currentValues` fields preserved, no caller
breakage. Optionally also echo `maxFlySpeed`/`jumpZVelocity` for full symmetry,
but the two avoidance/braking fields are the ones with no sibling reader and so
the highest-value to add.

**Workaround until then:** `propertiesSet` confirms the two fields were *written*
(by name); to confirm their *values*, read the CDO with
`property.get {objectPath:"/Game/.../BP_Foo.Default__BP_Foo_C", propertyName:"CharacterMovement.BrakingDecelerationWalking"|"CharacterMovement.AvoidanceWeight"}`
(cross-namespace; not signposted on the `set_ai_movement` wiki page).

## History
- `#1-initial-repro` `OPEN` reporter — Struggle-audit of a patrolling-guard end-to-end task (seed `ai.set_ai_movement`; finding lands on the seed itself). Task created `/Game/AI/Enemies/BP_GuardEnemy` (Character) and ran `set_ai_movement` with the full guard block (walk 240 / accel 1024 / braking 1800 / rotationRate 360 / orient true / RVO true / avoidanceWeight 0.6). Replayed verbatim against the live editor: the call succeeded, `propertyCount:7` and `propertiesSet` listed all 7 incl. `BrakingDecelerationWalking`/`AvoidanceWeight`, but `currentValues` echoed only `{maxWalkSpeed,maxAcceleration,rotationRateYaw,orientRotationToMovement,useRVOAvoidance}` — silently omitting `brakingDeceleration:1800` and `avoidanceWeight:0.6`. Independently confirmed both omitted writes DID persist via `property.get` on `Default__BP_GuardEnemy_C` (`CharacterMovement.BrakingDecelerationWalking` -> 1800, `CharacterMovement.AvoidanceWeight` -> 0.6000000238418579), so this is a partial-readback ergonomic gap, NOT a lost write or tool bug. The two dropped fields are precisely the ones with no sibling reader (`ai.get_ai_info` has no movement branch), so they are unconfirmable from any `ai`-namespace response without a cross-namespace `property.get` CDO read. Proposed the additive two-line `currentValues` mirror fix (`AIHandler.cpp:2765-2771`). Distinct namespace/method from the `character` `get_character_info` readback tickets (`F-character-info-no-friction-braking`, `E-character-readback-fallback-undocumented`); here the partial reader is the setter's own inline echo. (The attempt agent's own self-report misremembered `currentValues` as confirming only `maxWalkSpeed=240` — the real response confirms 5 of 7; the genuine, replay-verified gap is the 2 it drops. The agent's separate noted friction — `get_ai_info {controllerPath}` not surfacing `DefaultBehaviorTree` — is working-as-documented per the `ai.get_ai_info` wiki Notes, which prescribe `property.get` on the controller CDO; not a finding.)
