---
id: E-control-time-of-day-pitch-misreports-after-normalize
title: "environment.control.set_time_of_day echoes the pre-normalization solar pitch (e.g. pitch:165 for hour 17) but the directional light actually stores the normalized pitch (15), so a verify read-back contradicts the response"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [environment, environment-control, set-time-of-day, directional-light, pitch, rotation, normalization, readback, misreport, golden-hour]
---

# `environment.control.set_time_of_day` reports a `pitch` the actor never stores for afternoon/evening hours

`environment.control.set_time_of_day {hour}` rotates the level's directional
light (sun) and echoes the pitch it applied: `{hour, pitch, actor, ...}`. The
handler computes `SolarPitch = (hour/24)*360 - 90` and writes it straight onto
the actor rotation:

```cpp
// EnvironmentHandler.cpp:731-746
const float SolarPitch = (ClampedHour / 24.0f) * 360.0f - 90.0f;   // hour 17 -> 165
FRotator NewRotation = SunLight->GetActorRotation();
NewRotation.Pitch = SolarPitch;
SunLight->SetActorRotation(NewRotation);
...
Result->SetNumberField(TEXT("pitch"), SolarPitch);   // echoes 165
```

For any hour whose `SolarPitch` falls outside the canonical `[-90, 90]` pitch
range (i.e. roughly hours 12–24, the entire afternoon/evening), `SetActorRotation`
stores the **normalized** orientation — pitch is re-expressed into `[-90, 90]`
with yaw/roll flipped by 180° — so the value that actually lands on the
directional light differs from the `pitch` the response reports. The response
echoes the **pre-normalization Euler value the handler tried to set**, not the
value the actor holds afterward.

This is concretely misleading for the exact workflow the response invites: an
agent told to "confirm each lighting change took effect on the world's
directional light" reads the actor back and finds `pitch=15`, flatly
contradicting the RPC's `"pitch":165`. (The *lighting* result is plausibly
fine — a normalized pitch 15 with a 180° yaw flip is still a low sun — but the
echoed number is wrong, so the self-report can't be trusted as a verification.)

## Repro (verbatim, replay-confirmed against `mcp__editor-automation__call`)

Map `ExampleProjectWelcome`, actor
`/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.DirectionalLight_0`.

1. `environment.control.set_time_of_day {hour:17}`
   -> `{"hour":17,"pitch":165,"actor":".../DirectionalLight_0", ...}` (reports pitch **165**)
2. `actor.get_transform {actorName:".../DirectionalLight_0"}`
   -> `{"location":[0,10,0],"rotation":[15.000000000000018,-67.55410444594882,-108.81014357260482],"scale":[2.5,2.5,2.5]}`
   — actual stored pitch is **15**, not 165.
3. `property.get {objectPath:".../DirectionalLight_0", propertyName:"RootComponent.RelativeRotation"}`
   -> `{"value":[15.000000000000027,112.44589555405116,71.18985642739518], ...}`
   — same: pitch component **15** (the 180° flip shows in the other two axes:
   `-67.55 + 180 = 112.45`, `-108.81 + 180 = 71.19`).

Control (in-range pitch echoes correctly):

4. `environment.control.set_time_of_day {hour:12}` -> `"pitch":90`;
   `actor.get_transform` -> `rotation:[90, 41.256..., 0]` — first element **90**,
   matches. So the mismatch is specific to out-of-`[-90,90]` solar pitches
   (afternoon/evening hours), confirming a normalization-vs-echo gap, not a flat
   off-by-something.

## What it should do

Echo the value the actor **actually holds after the set**, not the value the
handler attempted. Either is a one-line change in `EnvironmentHandler.cpp`:

- Read the post-`SetActorRotation` rotation back and report **that** pitch:
  `Result->SetNumberField(TEXT("pitch"), SunLight->GetActorRotation().Pitch);`
  (or report the full applied `{pitch, yaw, roll}` so the 180° yaw flip is
  visible), instead of echoing the pre-normalization `SolarPitch` local.
- Alternatively, normalize `SolarPitch` for reporting the same way
  `SetActorRotation` does. The key is that the echoed `pitch` must equal what
  `actor.get_transform` / `property.get` return for the same actor, so the
  response is a usable self-verification.

## Not a duplicate of

- `E-set-time-of-day-no-suntime-readback` (OPEN) — that is the **`environment.build`**
  sky-sphere variant, whose complaint is the response echoes **no value at all**
  plus the `Sun height` readback property name is undocumented. This ticket is the
  separate **`environment.control`** directional-light variant, which **does**
  echo a `pitch` — the defect here is that the echoed value is **wrong** (misreports
  the normalized pitch), a distinct method and a distinct root cause.
- `E-environment-control-subnamespace-no-index-page` (OPEN) — missing wiki index
  page for the `environment.control` sub-namespace; a discoverability gap, orthogonal
  to this result-misreport.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed (golden-hour cinematic
  lighting task, REALISM mode, against `mcp__editor-automation__call`). The
  attempt agent's self-report claimed "hour 17 (sun pitch 165) ... on
  DirectionalLight_0" and that "each response referenced the live world actor it
  modified" — but reading the actor back shows the directional light stores
  pitch **15**, not the reported 165, because `SetActorRotation` normalizes the
  out-of-range `SolarPitch=165` (`EnvironmentHandler.cpp:731-746`) while the
  response echoes the pre-normalization local. Verbatim:
  `set_time_of_day {hour:17}` -> `"pitch":165`;
  `actor.get_transform` -> `rotation:[15.0, -67.55, -108.81]`;
  `property.get RootComponent.RelativeRotation` -> `[15.0, 112.45, 71.19]`.
  Control `hour:12` -> `"pitch":90` matches the stored `[90, ...]`, confirming the
  mismatch is specific to afternoon/evening hours whose solar pitch exceeds 90°.
  The sun-intensity (4.0) and skylight-intensity (0.6) sets in the same task
  landed correctly (`LightComponent.Intensity` read back 4 and 0.6 respectively),
  so this is isolated to the `set_time_of_day` pitch echo. Dedup (ripgrep over
  OPEN+closed; qmd unavailable): distinct from `E-set-time-of-day-no-suntime-readback`
  (the `environment.build` no-echo variant) and `E-environment-control-subnamespace-no-index-page`
  (missing index page). Fix: report the post-set actor pitch (or full applied
  rotation), not the pre-normalization `SolarPitch`.
- `#2-attempt-failed` `OPEN` developer — Auto-fix attempt reached NO-RESULT; reverted and NOT pushed (build/tests not green).
- `#3-fix` `IN-REVIEW` developer — Root-cause fix: `environment.control.set_time_of_day`
  now reads the directional light's rotation back AFTER `SetActorRotation` and echoes
  the value the actor ACTUALLY holds, instead of the pre-normalization `SolarPitch`
  local. `EnvironmentHandler.cpp:744-770` captures `AppliedRotation =
  SunLight->GetActorRotation()` post-set and emits `pitch`/`yaw`/`roll` from it (the
  full applied rotator, so the 180-deg yaw/roll flip is visible) — so the echoed
  `pitch` now equals what `actor.get_transform` / `property.get` return for the same
  actor (hour 17: pitch ~15, not 165). Files: `Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp`.
  Regression test: `PinWright.environment.control.set_time_of_day.EchoesStoredPitch`
  in `Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp` — spawns a
  directional light, invokes the real handler at hour 17, resolves the exact actor
  the response names, and asserts every echoed rotation component equals the actor's
  stored transform AND that the echoed pitch falls in the canonical `[-90,90]` range.
  A revert to echoing `SolarPitch` reports 165, which neither matches the stored ~15
  nor lies in `[-90,90]`, failing the test.
