---
id: B-step-frame-world-advance-constant
title: "editor.step_frame reports a fixed ~0.333 s worldSecondsAdvanced regardless of deltaSeconds — bit-identical across a 0.017 s and a 0.15 s request"
status: OPEN
severity: High
category: bug
tags: [editor, step_frame, pie, pause, fixed-timestep, world-tick, timing, capture, hud, silent-wrong-data]
encounters: 3
costly: 2
lastSeen: 2026-09-07T08:39:00Z
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
