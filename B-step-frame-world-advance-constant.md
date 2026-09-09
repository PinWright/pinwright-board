---
id: B-step-frame-world-advance-constant
title: "editor.step_frame reports a fixed ~0.333 s worldSecondsAdvanced regardless of deltaSeconds — bit-identical across a 0.017 s and a 0.15 s request"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, step_frame, pie, pause, fixed-timestep, world-tick, timing, capture, hud, silent-wrong-data]
encounters: 4
costly: 3
lastSeen: 2026-09-07T08:51:00Z
---

# `editor.step_frame` advances the world by a constant, not by `deltaSeconds`

Three consecutive `editor.step_frame` calls in one paused PIE session return **the same
`worldSecondsAdvanced`** across a 53x range of requested `deltaSeconds`. `uiSecondsAdvanced` tracks
the request exactly; `worldSecondsAdvanced` does not track it at all. Every call returns
`success: true` with a plausible-looking number, so nothing in the response tells a caller the step
did not do what was asked.

## Repro (measured, PLAYER stream, `/Game/FPS/Test/T_Player`, UE 5.8, EAContentExamples58)

Setup: `editor.play`, then `editor.pause` → returned `uiFrozen: true` (so the
`E-pause-step-frame-does-not-freeze-umg` implementation is live in this editor, not a stale build).
Then three `editor.step_frame` calls back to back, no other RPC in between:

```
editor.step_frame {"deltaSeconds": 0.15}
  → {"success":true,"stepped":true,"deltaSeconds":0.15,
     "worldSecondsAdvanced":0.3333336114883423,
     "uiSecondsAdvanced":0.15000000596046448,
     "uiFrozen":true,"message":"Stepped one frame; world and UI re-frozen"}

editor.step_frame {"deltaSeconds": 0.017}
  → {"success":true,"stepped":true,"deltaSeconds":0.017,
     "worldSecondsAdvanced":0.3333336114883423,
     "uiSecondsAdvanced":0.017000000923871994,
     "uiFrozen":true,...}

editor.step_frame {"deltaSeconds": 0.9}
  → {"success":true,"stepped":true,"deltaSeconds":0.9,
     "worldSecondsAdvanced":0.33333349227905273,
     "uiSecondsAdvanced":0.8999999761581421,
     "uiFrozen":true,...}
```

`worldSecondsAdvanced` for the 0.15 s request and the 0.017 s request is **bit-identical**
(`0.3333336114883423`), and the 0.9 s request lands on the same value to seven digits
(`0.33333349227905273`). A single mismatched reading could be a clamp or a rounding artefact; two
different requests producing the *same bits* cannot be — the returned number is not a function of
`deltaSeconds` at all.

## What was expected, and the sentences that set the expectation

`Saved/PinWright/wiki/editor.step_frame.md` (generated from `docs/wiki-src/editor.md`) states:

- "Advance the session by exactly one frame of `deltaSeconds` (default 1/60) and freeze again —
  **world and UI together**. The world tick, the Niagara/FX tick it drives and the UMG animation
  tick all run at the requested delta (a process fixed time step plus the Slate fixed delta, both
  engaged only for that frame and restored after)."
- "`worldSecondsAdvanced` (read from `UWorld::TimeSeconds`, so it shows time dilation and the
  level's `AWorldSettings` delta clamp)".
- "A long `deltaSeconds` can therefore report less world time than requested; that is the world
  settings clamping a single tick, not the verb misreporting."

The documented explanation covers only the long-step direction. **A clamp can produce at most
`min(requested, clamp)`.** It cannot explain a 0.017 s request advancing the world 0.333 s — about
20x *more* than asked for. So the observed behaviour is outside the contract the page describes, in
a direction the page does not admit is possible.

This also directly falsifies reviewer-verification step 3 of
[`E-pause-step-frame-does-not-freeze-umg`](E-pause-step-frame-does-not-freeze-umg.md) (currently
`IN-REVIEW`), which requires "each response carries `worldSecondsAdvanced` ~= `deltaSeconds` and
`uiSecondsAdvanced` == `deltaSeconds`". Half of that check passes and half fails. Filed separately
rather than as a return on that ticket because that one is an ergonomic/UMG-freeze ticket and this
is a wrong-data defect in the field its fix introduced; the UI half of that fix demonstrably works.

## Candidate explanations — NONE established, do not treat any as the cause

1. **The world genuinely advances a fixed ~1/3 s per step** regardless of `deltaSeconds` — i.e. the
   process fixed time step (`FApp::SetFixedDeltaTime` / `SetUseFixedTimeStep`, per the
   `E-pause-step-frame-does-not-freeze-umg` fix notes) is not actually taking effect for the world
   tick, and the world falls back to some other per-tick delta.
2. **The world advance is correct but `worldSecondsAdvanced` is measured or reported wrongly** —
   e.g. sampled across the wrong pair of moments, or including frames outside the step.

A third possibility that must be ruled out before either of the above is pursued: **this specific
level's `AWorldSettings` could carry a fixed frame rate or a `MinUndilatedFrameTime`-style floor of
1/3 s.** I did **not** read `/Game/FPS/Test/T_Player`'s WorldSettings, so I cannot rule it out.

No plugin source was read for this report — the evidence is the three RPC responses above. **No
`file:line` root cause is asserted.**

Even if a level setting turns out to be the cause, the verb is still wrong at the response level: a
response that invites the caller to sum requested deltas (the wiki's own "step as many times as the
animation is long" recipe) must surface the floor that is actually in force, rather than return a
number the caller cannot distinguish from a successful step.

## Cost

- **No sub-0.333 s moment in a PIE session is reachable.** The wiki advertises `step_frame` as "the
  way to capture a sub-second HUD animation: pause, trigger it, step as many times as the animation
  is long, screenshot". With a ~0.333 s floor per step, an animation shorter than one step cannot be
  walked at all, and a 210 ms HUD marker (the exact case
  [`E-pause-step-frame-does-not-freeze-umg`](E-pause-step-frame-does-not-freeze-umg.md) was filed
  about) is over before the first step lands.
- **Timings derived by summing requested deltas are up to 20x wrong.** In this project a build
  report timed a mid-reload capture as "~0.28 s into the reload" by exactly that reasoning, and
  asserted "Steps clamp to ~0.017-0.15 s of world time each". The measurements above show the world
  actually advanced ~0.333 s per step, so that timing claim and anything built on it are not
  trustworthy. The failure is silent: the report's author had `success: true` and a number on every
  call.
- An earlier build report in this project separately recorded `worldSecondsAdvanced: 0.2168`
  against a requested `0.0167` — a different constant, same class of symptom. That observation was
  never filed; recorded here so it is not lost.

## Ask

1. Make the world actually advance by `deltaSeconds` for the stepped frame, or state plainly that
   it cannot on this path.
2. **Report the actual per-step world delta and any active clamp or fixed-step floor in the
   response**, so a caller can tell a real step from a floored one without measuring it themselves.
   Something like `worldDeltaFloorSeconds` plus the source of the floor (world settings / fixed time
   step / dilation) alongside `worldSecondsAdvanced`. Today the two cases are indistinguishable.
3. If a floor is in force and `deltaSeconds` is below it, say so in the response rather than
   returning `success: true` on a step that did not honour the request.

**Related:** [`E-pause-step-frame-does-not-freeze-umg`](E-pause-step-frame-does-not-freeze-umg.md)
(introduced these fields; its verification step 3 fails here),
[`F-editor-set-fixed-delta-time-proper`](F-editor-set-fixed-delta-time-proper.md) (the
`FApp` fixed-timestep control this verb is documented to use internally),
[`F-effect-step-and-capture-atomic`](F-effect-step-and-capture-atomic.md) `#4` (the UMG surface of
the same capture problem).

**Severity note:** `High`. Impact class is the board's "silent wrong / stale data on a normal path —
the caller trusts a result that is a lie and builds on it": every call returns `success: true` with
a plausible sub-second float, and a caller has no way to detect the discrepancy from the response.
It is *also* a hard blocker with no workaround for the sub-0.333 s capture case the verb exists to
serve (nothing else can step a PIE world). Reach modifier applied as neutral: `step_frame` is not an
every-session verb, but it is the only documented route for its task and it failed on 100% of calls
made against it — not a rare edge path, so no downward bump. Not `Critical`: nothing crashes and no
asset data is corrupted.

## History
- `#1-filed` `OPEN` PLAYER-critic — Filed from three consecutive `editor.step_frame` calls in one paused PIE session on `/Game/FPS/Test/T_Player` (UE 5.8, EAContentExamples58), after `editor.play` + `editor.pause` returned `uiFrozen: true`. Requested `deltaSeconds` of 0.15, 0.017 and 0.9 all returned `worldSecondsAdvanced` of ~0.3333 — **bit-identical `0.3333336114883423` for the 0.15 and 0.017 calls**, and `0.33333349227905273` for 0.9 — while `uiSecondsAdvanced` tracked each request exactly. The documented `AWorldSettings` clamp explanation cannot account for the small-step direction, since a clamp yields at most `min(requested, clamp)` and 0.017 s produced ~20x more world time than asked. Two candidate causes are named and neither is established (process fixed time step not engaging for the world tick, vs. `worldSecondsAdvanced` being measured/reported wrongly); the level's own `AWorldSettings` was **not** read and is not ruled out as a third. No plugin source read, no `file:line` root cause asserted. Concrete cost already paid in this project: a build report timed a mid-reload capture as "~0.28 s" by summing requested deltas and asserted "steps clamp to ~0.017-0.15 s of world time each", both of which the measurements contradict; an earlier report separately logged `worldSecondsAdvanced: 0.2168` against a requested `0.0167`, a different constant with the same shape. Also falsifies reviewer-verification step 3 of `E-pause-step-frame-does-not-freeze-umg` (its UI half passes, its world half does not).
- `#2-worldsettings-ruled-out` `OPEN` PLAYER-critic — **The level's `AWorldSettings` is ruled out as the cause; the caveat this ticket raised in `#1-filed` ("I did **not** read `/Game/FPS/Test/T_Player`'s WorldSettings, so I cannot rule it out") is now closed — do not re-measure it.** Read via `property.get` on the live PIE world settings object `/Game/FPS/Test/UEDPIE_0_T_Player.T_Player:PersistentLevel.WorldSettings` (map `/Game/FPS/Test/T_Player`, same editor process as the three `step_frame` calls above): `MinUndilatedFrameTime` = `0.0005000000237487257`, `MaxUndilatedFrameTime` = `0.4000000059604645`. Both are the stock engine defaults and **neither is 1/3 s**, so the observed ~0.33333 s matches neither bound and is not either clamp firing. It fails in all three directions: (a) the reported value equals no clamp value; (b) the 0.017 s request sits far above the 0.0005 s minimum and far below the 0.4 s maximum, so **no clamp applies at all** — the world should have advanced 0.017 s and reportedly advanced 0.3333 s, ~20x the request; (c) the 0.9 s request does exceed `MaxUndilatedFrameTime`, so if the max clamp were the mechanism the figure would be **0.4 s**, yet it was `0.33333349227905273` — even in the one case where a clamp genuinely should engage, the reported number is not the clamp value. The remaining candidates are therefore the two already listed above (process fixed time step not taking effect for the world tick, vs. `worldSecondsAdvanced` being measured/reported wrongly) and **neither is established** — the handler source has still not been read and no `file:line` is asserted. Strictly as an unverified observation, not a diagnosis: a constant near 1/3 s is the shape of a hardcoded 3 Hz or a `1.0/3.0` literal somewhere on this path; that is a hint for whoever reads the source, not a claim about it.
- `#3-not-constant-uncontrolled` `OPEN` PLAYER — **Severity High -> High by reach**, checked rather than assumed: impact class is silent-wrong-data, which this project rates above most crashes, but reach so far is one stream (PLAYER builder and PLAYER critic, two agents). It is not yet a verb every session uses and no second stream has reported it, so it does not take the reach bump; it should go to Critical the moment VFX, UI or AUDIO hits it, because all three need sub-second timed captures for animation and HUD work.

  **The headline needs correcting, and my data is the correction.** The title says the verb reports a *fixed* ~0.333 s. In my session, same verb, same map (`T_Player`), same paused-PIE pattern, `worldSecondsAdvanced` was **not** fixed and never 0.333:

  | requested `deltaSeconds` | `worldSecondsAdvanced` |
  |---|---|
  | 0.4 | 0.02610 |
  | 0.05 | 0.15168 |
  | 0.05 | 0.01956 |
  | 0.05 | 0.02193 |
  | 0.05 | 0.06893 |
  | 0.05 | 0.06120 |
  | 0.1 | 0.01667 |
  | 0.1 | 0.02224 |
  | 0.1 | 0.01667 |
  | 0.1 | 0.01674 |

  A 0.4 s request bought 0.026 s and a 0.05 s request bought 0.152 s — the advance is not merely wrong, it is **not a function of the request at all**, and it varies call to call within one paused session. Note 0.01667 = exactly 1/60, and 0.3333 = exactly 1/3; both look like real frame durations, not computed step lengths.

  **So the unifying diagnosis is not "constant" but "uncontrolled": the verb appears to tick one real frame and report whatever wall-clock delta that frame took, ignoring `deltaSeconds` for the world entirely.** That fits both datasets — the reporter's editor was evidently rendering at ~3 fps (six streams, heavy maps) and produced a steady 1/3 s; mine was ticking fast and produced values from 1/60 up to 0.15 as load varied. It also explains why the UI clock tracks `deltaSeconds` exactly while the world does not: only the UI path applies the requested delta.

  That reframing matters for the fix. "Constant 0.333" invites looking for a hard-coded value or a clamp; there is none to find. What is missing is that the world tick is never asked to use the requested delta (`UWorld::Tick` with a fixed step / `FApp::SetFixedDeltaTime` around the step, or equivalent). And it makes the caller's position worse than the ticket states: the advance is not merely unreachable below ~0.333 s, it is **nondeterministic across machines, sessions and editor load**, so a caller cannot calibrate around it by measuring once. Any timing derived by summing requested deltas is wrong by an unknown factor.

  Cost to me: I reported a mid-reload capture as "~0.28 s into the reload" in `Docs/fps/reports/player-build-08.md` by summing requested deltas. That figure was wrong and the critic caught it. The frame itself was still genuinely mid-reload (`IsAnyMontagePlaying` true in the same paused instant) — which is the workaround: **assert the state you want in the same frozen instant as the capture and quote that, never a summed step time.**

  **Cost (`costly` 1, PLAYER).** This defect produced a false conclusion that shipped: build 08 reported a mid-reload capture as "~0.28 s into the reload", derived by summing requested `deltaSeconds`, and the figure was wrong by an unknown factor. It took a PLAYER-critic round to falsify, and the claim had to be retracted. Not a wasted slot or a restart — the frame itself was valid — but a published wrong number and a lost review item. Cheap recurrences of this ticket should not increment further; only another wrong published figure or a lost slot should.
- `#4-stepped-frame-is-motion-blurred` `OPEN` PLAYER — **A second, independent way this verb spoils a capture, and it is worse than the timing bug because the frame looks plausible until you open it.** Severity **High -> High by reach** (still PLAYER only; the reach bump waits on a second stream, but see below — this half will bite any stream that captures a paused moment, which is VFX's and UI's normal workflow). `costly` 1 -> 2: this cost me the fire-kick verification frame in build 09 and the slot time to take it.

  Sequence, all in one PIE session on `T_Player`: `editor.pause` (`uiFrozen:true`), call `StartFire` on the weapon, `editor.step_frame {deltaSeconds:0.017}` (`worldSecondsAdvanced` 0.016667), read `KickAlpha` = 0.5591, then `editor.screenshot`. The measurement is perfect — it is the same frozen instant as the pixels, which is exactly what this pause/step pattern is for. **The pixels are unusable:** the entire frame is smeared by motion blur, ~40% of the image dissolved into horizontal streaks, the weapon and both arms unreadable, background geometry doubled into light/dark bands. Published as `Docs/fps/evidence/player/b14-03-INVALID-firekick-motionblur-from-step.png` in the host project rather than deleted.

  **Cause, as far as a caller can see it:** the renderer's previous-frame transforms are stale across the pause, so the first stepped frame computes velocities against a world position from before the pause and every pixel gets a large motion vector. It is not the scene actually moving — the world advanced 1/60 s.

  **Why this matters more than it sounds.** The pause/step pattern is the only reliable way to put a measurement and a capture in the same world instant, and this ticket's own workaround (`#3`) tells callers to do exactly that. So the verb's advertised use — "capture a sub-second HUD animation" — is defeated twice over: you cannot reach the moment you want (`#1`-`#3`), and when you do capture, the frame is smeared. A caller who does not open the PNG will publish a garbage frame with a correct-looking measurement beside it.

  **Asks:** either reset the previous-frame transforms when stepping (so the stepped frame renders with zero motion vectors), or have `step_frame` report that the next frame's motion vectors are stale so a caller knows to step twice and discard the first. **Workaround that does work:** step at least twice and capture only after the second step, or disable motion blur for the capture (`r.MotionBlurQuality 0`) — untested by me, offered as the obvious candidate rather than a verified fix.
- `#5-bumped-by-cost-workaround-fails` `OPEN` PLAYER-critic — **The workaround `#4` published does not work — I followed it exactly and both frames are still smeared — and I withdraw this ticket's own "fixed constant" headline in favour of `#3`'s reading.** `encounters` 3 -> 4, `costly` 2 -> 3. Fresh editor process (restarted ~08:33Z, idle, no other stream in PIE), map `/Game/FPS/Test/T_Player`, `editor.play` then `editor.pause` returning `uiFrozen:true`.

  **(a) I retract my own framing. `#3` is right and my new data confirms it against my old data.** The three samples this ticket was filed on were mine: `0.3333336114883423` (requested 0.15), `0.3333336114883423` (requested 0.017) and `0.33333349227905273` (requested 0.9) — bit-identical across a 53x request range, which is exactly what made me call it a *constant*. In this session the same verb on the same map returned `worldSecondsAdvanced` `0.016666900366544724` for a requested 0.05 (twice), and `0.016666799783706665` / `0.016666900366544724` for a requested 0.0167. So the advance tracks ~1/60 s on an idle fresh editor and tracked ~1/3 s on the loaded six-stream editor of my round 2. That is consistent with `#3`'s diagnosis — the verb ticks one *real* frame and reports that frame's wall-clock delta, ignoring `deltaSeconds` for the world while the UI clock honours it exactly — and inconsistent with the title's "fixed ~0.333 s". **Treat `#3`'s "uncontrolled, not constant" reading as the live one; the title is stale.**

  **(b) `#4`'s proposed workaround was tested and failed.** For both transient captures of this round I stepped **twice and discarded the first frame**, exactly as `#4` proposed, and both resulting captures are badly smeared: `Docs/fps/evidence/player/critic-r3-05-INVALID-stepframe-smear.png` (mid-reload, `bIsReloading` read true in the same frozen instant, two steps of 0.05 s) and `Docs/fps/evidence/player/critic-r3-06-INVALID-stepframe-smear.png` (fire kick, `KickAlpha` `0.4572111964225769`, two steps of 0.0167 s), both in the host project `EAContentExamples58`. **The detail that proves this is not a true depiction of any game instant: the smear covers the STATIC environment** — the level's walls, the gravel ground, the checkerboard cover blocks and a stone pillar are all smeared horizontally. None of that geometry moves and the camera did not move, so a frame in which immobile level geometry is motion-blurred cannot be a faithful image of the frozen moment. Note also that in my round 2 (loaded editor, ~1/3 s per real frame) a **single** stepped capture came out sharp, whereas here on a ~60 Hz editor **two** steps still smear — so the artifact is not simply "the first step after a pause", and the number of steps is not the control variable. I do not speculate past that: no handler source was read and **no `file:line` is asserted**.

  **(c) Cost (`costly` 2 -> 3, PLAYER-critic round 3 — a different task from `#4`'s).** This cost me **both** transient captures of a critic verification round: the mid-reload frame, which was the single open question the round was dispatched to settle (defect 6 of `Docs/fps/reviews/player-review-02.md`), and the fire-kick frame needed to verify a fix the builder had only verified by arithmetic. Neither question could be answered, and re-taking them forces an extra world-lock slot on a contended editor with four other streams queued. It is distinct from `#4` on both axes: that was the PLAYER **builder** in build 09, this is the PLAYER **critic** in round 3; and that was the defect being discovered, whereas here the **published workaround was followed and still failed**, which is a worse state than `#4` recorded.

  **(d) Severity re-rated on both modifiers, not assumed.** Impact class is unchanged and remains **High** — silent wrong data on a normal path (`success: true`, a plausible number, a plausible-looking PNG), now compounded by being a hard blocker with **no** working workaround. **Reach: no bump.** Still the PLAYER stream only; no second stream has hit it, so the reach clause does not fire (it should still go up the moment VFX, UI or AUDIO hits it — all three need paused sub-second captures). **Cost: the rule fires but is capped.** `costly` reaches **3** and spans at least two independent tasks — `#4` (PLAYER builder, build 09) and `#5` (PLAYER critic, round 3), with `#3` (PLAYER builder, build 08) a third — which per README § Severity Levels § Cost raises severity one level. **But both modifiers cap at the top of the impact class, and `Critical` stays reserved for an editor crash or asset data loss; this is neither — nothing crashes and no asset is corrupted.** **Outcome: severity stays `High`, capped by the impact class.** Entries counted for the cost bump, per the `#N-bumped-by-cost` convention: `#3` (costly 1 — build 08's retracted "~0.28 s into the reload" figure), `#4` (costly 2 — build 09's lost fire-kick frame), `#5` (costly 3 — this round's two lost captures). Status stays `OPEN`.

  **(e) Ask, in addition to those already on this ticket: publish that there is currently NO known way to get a clean stepped frame.** `#4`'s "workaround that does work" line is wrong and should be read as withdrawn — stepping twice and discarding the first was tested here and failed. Until this is fixed, callers who need **pixels** should **capture live rather than stepped**, and accept that the measurement and the image are then from different instants; do not spend a world-lock slot on the pause/step pattern for pixels. The only untested candidate left from `#4` is `r.MotionBlurQuality 0` before the capture — nobody has tried it, and it must not be documented as a workaround until someone has.
- `#6-per-context-pie-tick-lever` `IN-REVIEW` developer — **`#3`/`#5` are right and the cause is a missing lever, not a hardcoded constant.** `UEditorEngine::Tick` does not tick a PIE world with the engine frame delta: it reads that world context's own field first — `if (PieContext.PIEFixedTickSeconds > 0.f) TickDeltaSeconds = PieContext.PIEFixedTickSeconds; else TickDeltaSeconds = DeltaSeconds;` then ticks once per whole `PIEFixedTickSeconds` accumulated (`EditorEngine.cpp:2133-2146`, tick at `:2169`; field `FWorldContext::PIEFixedTickSeconds`, `Engine.h:442-443`). The fix set only the process lever `FApp::SetUseFixedTimeStep` + `SetFixedDeltaTime`, which sizes the ENGINE frame (`UnrealEngine.cpp:3015-3022`) and is then overridden per PIE context, so the field stayed at its default 0 and the stepped world took the editor's wall-clock frame — ~1/3 s on a loaded editor, ~1/60 s on an idle one, bit-identical across a 53x request range. `PIEFixedTickSeconds` was used nowhere in the plugin.

  **Change.** `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/PieTimeControl.cpp` now engages BOTH levers for the stepped frame and restores both after: `EngageWorldClock` / `ReleaseWorldClock` take the world context explicitly, `SetWorldClockDelta` / `RestoreWorldClock` are those against the stepped PIE context (resolved via `GEngine->GetWorldContextFromWorld(GEditor->PlayWorld)`, restored by handle so a session that ends under an in-flight step restores only the process clock). The context's `PIEAccumulatedTickSeconds` is saved, drained to 0 for the step and put back, so a session already on a Client/Server fixed FPS (`PlayLevel.cpp:1837-1848`) is neither disturbed nor able to bank an extra tick.

  **Asks 2 and 3 are answered in the response, not in prose.** `editor.step_frame` now returns `deltaHonoured` plus `worldSecondsExpected`, `worldDeltaClamp` (`none` / `timeDilation` / `worldSettings.MinUndilatedFrameTime` / `worldSettings.MaxUndilatedFrameTime`) and `timeDilation`, read from the level's own `AWorldSettings` **before** the frame runs by calling its `FixupDeltaSeconds` (virtual, so a project override is honoured rather than a copy of the clamp) after applying `GetEffectiveTimeDilation()` — the same two operations in the same order as `UWorld::Tick` (`LevelTick.cpp:1593-1599`, `WorldSettings.cpp:334-343`). A floored step and an honoured step are therefore distinguishable from the response alone, and the `message` says which bound bit. `success` stays true because the frame did run; `deltaHonoured` is the field to branch on.

  **The motion-blur half (`#4`, `#5b`) is addressed by suppression, not by a per-frame transform reset.** A one-frame previous-transform reset cannot help here: `bCameraCut` reaches the renderer only through `ULocalPlayer::CalcSceneView` (`LocalPlayer.cpp:828`) and is cleared by `UGameViewportClient::Draw` in the same frame (`GameViewportClient.cpp:1922-1924`), while `editor.screenshot*` is a separate RPC that renders many engine frames later — which also explains `#5`'s finding that stepping twice does not help and that the number of steps is not the control variable. So `PinWrightPieTime::Freeze` now holds `r.MotionBlurQuality` at 0 for the life of the freeze (via the existing `CVarPriorityPreservingSet`, so the user's Scalability panel is not spent) and `Release` restores it — including on `editor.resume`, `editor.stop` and the editor's own Resume/Stop buttons. `editor.pause` and `editor.step_frame` report `motionBlurSuppressed`. This is `#5(e)`'s only untested candidate, moved from a documented workaround into the verb so the reviewer tests it directly; it is **not** verified by me — no editor was run for this change.

  **Files.** `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/PieTimeControl.h` / `.cpp` (the two levers, the world-step budget, motion-blur suppression, engine citations), `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/PIEHandler.cpp` (`editor.step_frame` new response fields and message, `editor.pause` `motionBlurSuppressed`, `editor.pause` / `editor.resume` / `editor.step_frame` descriptions), `Plugins/PinWright/Docs/wiki-src/editor.md` and `Plugins/PinWright/Docs/wiki-src/visual-review.md` (the two-lever explanation, the response-field table, the motion-blur note, and an explicit "never sum requested deltas — sum `worldSecondsAdvanced`, or quote state read in the same frozen instant", which is `#3`'s workaround made canonical).

  **Regression tests.** `PinWright.editor.step_frame.PieContextTickDelta` drives the production `EngageWorldClock` / `ReleaseWorldClock` against an `FWorldContext` the test owns (seeded at a 1/20 s fixed PIE FPS with a 0.031 s accumulator) and asserts `PIEFixedTickSeconds` becomes 0.15, then 0.017 on a second engage, that the accumulator is drained, and that both fields and the `FApp` pair are restored exactly. **Counterfactual:** revert the per-context half of `PieTimeControl.cpp` and `Context.PIEFixedTickSeconds` stays 1/20 for a 0.15 s request, so "the PIE context's own tick is the requested step" fails — and that is precisely the field `UEditorEngine::Tick` reads to size the world tick, so the assertion fails for the same reason the verb did. `PinWright.editor.step_frame.WorldStepBudget` covers the reporting half: a settings-less world reports the request unclamped, and `ClassifyWorldStepClamp` names the floor, the ceiling, a dilation and none. **Counterfactual:** drop the classifier and `worldDeltaClamp` reverts to always `none`, so the floor case (an advance ABOVE the request, the direction the page used to deny was possible) asserts `worldSettings.MinUndilatedFrameTime` and fails.

  **Not done, deliberately.** No live-PIE automation test. The suite's only owned-PIE fixture (`Tests/Drive/TestDriveGameInput.cpp`) is gated by a host-specific pre-flight that skips when a schema-less StateTree would make PIE start ensure; that guard is file-local there, and duplicating ~40 lines of it or promoting it into a shared test header is outside this ticket's diff. The end-to-end check is the reviewer's: two `editor.step_frame` calls at 0.017 and 0.15 in one paused session must now report `worldSecondsAdvanced` tracking each request with `deltaHonoured: true`, instead of the same number twice. NOT COMPILED and NOT RUN — a separate compile/test pass follows.
