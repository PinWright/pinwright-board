---
id: F-effect-step-and-capture-atomic
title: "No way to capture a sub-second Niagara effect: the editor world ticks in real time between RPCs, so activate -> advance_simulation -> capture_open_level always photographs a dead effect"
status: OPEN
severity: High
encounters: 4
category: feature-request
tags: [effect, niagara, umg, hud, pie, slomo, time-dilation, render, capture, advance_simulation, activate_niagara, timing, vfx, visual-review, state-sampling, game-time, weapons, cross-stream]
---

# A three-call capture cannot photograph anything shorter than the gap between two RPCs

## Problem

The documented way to prove a Niagara effect looks right is to place it in a level and
capture the level (`niagara.add_emitter` notes, `visual-review.md`, and the standing rule
that a Niagara asset preview must never be captured with `closeAfterCapture`). In practice
that is three separate RPCs:

```
effect.activate_niagara   { systemName, reset: true }
effect.advance_simulation { systemName, deltaTime: 0.005, steps: 8 }   // t = 40 ms
render.capture_open_level { ... }
```

**The editor world keeps ticking in real time between those calls.** Seconds of wall clock
pass between one RPC and the next, and `render.capture_open_level` adds its own settle pass
on top (`warmup.settleMs` was 33-160 ms across my captures). So by the time the shutter
opens, a combat effect is long dead and the frame shows an empty scene.

Measured, on UE 5.8 / EAContentExamples58, `/Game/FPS/VFX/NS_Impact_Concrete` placed in
`/Game/FPS/Test/T_VFX`:

- Sequence above, then capture: nothing in frame. `niagara.validate` on the same system
  immediately afterwards reported `componentActivation: "none_active"`, `componentCount: 1`,
  `isActive: false` — the component had finished and deactivated before the capture.
- The same three calls with world time frozen first: the effect is in frame, correctly
  positioned, at the exact step count requested.

Nothing in the response of any of the three verbs indicates the problem. `activate_niagara`
answers `active: true` (true when measured, stale a second later), `advance_simulation`
answers `{success: true, steps: 8}`, and `capture_open_level` answers a perfectly healthy
frame with `blank: false` — because the scene *is* there, only the effect is not. The
failure is silent and reads exactly like "the effect does not render", which is what I
concluded first and spent several captures disproving.

## Why the obvious readings are wrong

- It is not `autoDestroy`: the actors were spawned with `autoDestroy: false`.
- It is not the effect being broken: the identical asset renders correctly once time is
  frozen, and `niagara.validate` reports `valid: true`, `dataInterfaceCheck: "consistent"`.
- It is not the capture's settle pass alone: the settle is 33-160 ms, while the gap between
  RPCs is seconds. Both contribute; the RPC gap dominates.
- `advance_simulation` is not at fault — it does exactly what it says. The problem is that
  nothing holds the world still around it.

## Workaround (works, but should not be the answer)

`misc.set_game_speed { speed: 0.0001 }` before the sequence, then `1.0` afterwards. World
time is effectively frozen, `effect.advance_simulation` becomes the only clock, and stepping
is deterministic and repeatable — I captured the same impact at 20 ms, 40 ms, 100 ms and
600 ms and got four distinct, correct frames.

Three problems with it as the recommended route:

1. It is global world state in a **shared** editor. Every other agent's PIE, animation and
   simulation is frozen for as long as it is set, and if the setting agent dies between the
   two calls the world is left at 1e-4 speed with nothing to say why.
2. `speed: 0` is not usable (it reads as "no dilation" in places), so `1e-4` is a magic
   number a caller has to know.
3. It is documented nowhere near the capture verbs. `misc.set_game_speed`'s page is about
   slow-motion; nothing in `render.*`, `effect.*` or `visual-review.md` mentions that a
   short effect cannot otherwise be photographed.

## Requested

Either of these closes it; the first is much better:

- **An atomic step-and-capture verb**, e.g.
  `effect.capture_at { systemName, atSeconds, deltaTime, <capture args> }` — freeze or
  scope world time, reset and activate the system, step it to `atSeconds`, capture, restore.
  One RPC, no shared-state window, deterministic. A `times: [0.02, 0.1, 0.6]` variant
  returning several PNGs would make effect review a single call, which is what this job
  actually needs: an effect is judged across its life, not at one instant.
- **Failing that**, a scoped `freezeWorldTime: true` argument on `render.capture_open_level`
  (restored on every exit path, like the existing `viewMode` and `exposure` parameters
  already are), plus a line in `visual-review.md` and on the `effect.*` pages stating that a
  sub-second effect cannot be captured without it.

Also worth having regardless: `effect.advance_simulation` and `effect.activate_niagara`
should report the component's `isActive` **and** its particle count, so a caller can tell
"stepped a live effect" from "stepped a corpse" without a separate `niagara.validate`.

severity rationale: impact = the documented visual-verification path silently produces empty
frames for the entire class of assets it exists to verify (combat VFX are all sub-second),
and the failure mimics a content bug so the caller debugs the wrong thing x reach = any
agent verifying any short effect -> High.

## History

- `#1-filed` `OPEN` reporter — Hit while verifying the FPS VFX package on UE 5.8 / EAContentExamples58 in a shared multi-agent editor. Placed 14 combat systems in `/Game/FPS/Test/T_VFX` and could not photograph any of them: eight consecutive captures of muzzle flashes and impacts came back as correct, well-exposed, non-blank frames of an empty scene. Diagnosed by calling `niagara.validate` immediately after the capture and reading `componentActivation: "none_active"` / `isActive: false` off the placed component, which established the effect had completed rather than failed to render. Confirmed by freezing world time with `misc.set_game_speed {speed: 0.0001}` and re-running the identical three calls: the impact then appeared, and stepping it to 20/40/100/600 ms produced four distinct correct frames that let me identify three real content defects I had been unable to see. Note the diagnostic value went both ways — before the freeze I had already wrongly concluded that a healthy `NS_Impact_Concrete` "renders nothing". No plugin source read; evidence is the RPC responses, the `componentActivation` block, and the before/after captures under `Saved/PinWright/fps/vfx/`.
- `#2-same-root-cause-hit-from-the-state-sampling-side-by-another-stream` `OPEN` reporter — **The scope of this request is wider than captures, confirmed independently by a second stream on the same day.** The WEAPONS stream lost about an hour to a spread bug that did not exist: it read a weapon's spread value roughly 1.5 seconds of wall clock after firing and reasoned about it as 1.5 seconds of game time. The world had actually advanced **18.2 seconds** of game time in that gap — the editor ticks through RPC latency and agent turnaround, not only through deliberate sleeps — so a value that had long since decayed to its floor read as a recovery bug in the weapon. Same mechanism as `#1`, different symptom: `#1` is "the thing I wanted to photograph is already dead", this is "the number I read is from a much later moment than I think".
  That makes the missing capability broader than a capture verb. Any agent sampling *any* time-dependent state over MCP — spread bloom and recovery, recoil recentring, an animation's elapsed time, a decay curve, a cooldown — is reading it at an unknown and much later game time than the call sequence implies, with nothing in any response reporting the game time the value belongs to. Two streams hit it the same day from opposite ends and neither could see it from the RPC responses alone; both diagnosed it only by measuring the world afterwards.
  Two additions to the ask in `#1`, both cheap relative to the class of bug they close:
  1. **Report game time on time-dependent reads.** A `worldTimeSeconds` (and ideally `deltaSinceLastCall`) field on `effect.*`, `actor.describe`, `property.get` and the capture verbs would have let either of us spot the discrepancy in one response instead of an hour of wrong-tree debugging. It costs a float.
  2. **Scope the freeze beyond captures.** `#1` asks for an atomic step-and-capture; the same primitive is wanted for step-and-*read*. A scoped "hold world time for the duration of this call" — or a documented, supported freeze/restore pair safe to use in a shared editor — serves both. The current workaround, `misc.set_game_speed {speed: 0.0001}` around the sequence and `1.0` afterwards, is global state in a process shared by six agents: it silently freezes everyone else's PIE and simulation, and if the setting agent dies mid-sequence the world is left at 1e-4 with nothing to say why. It works, it is what both of us used, and it should not be the recommended route.
  No new source read. Evidence is `#1`'s measurements plus the WEAPONS stream's own report of the 18.2 s discrepancy and the hour it cost. Raising `encounters` to 2; severity unchanged at High, though the reach argument is now stronger than `#1` stated — it is not only agents verifying short effects, it is any agent reading time-dependent state at all.
- `#3-freeze-works-but-it-breaks-the-documented-exposure-recipe` `OPEN` reporter — Third encounter, VFX critic review, same editor, 2026-09-02. Confirming `#1`'s workaround from an independent run: `misc.set_game_speed {speed: 0}` (which clamps to and honestly reports `actualTimeDilation: 1e-4`) followed by per-system `effect.activate_niagara {reset:true}` -> `effect.advance_simulation {deltaTime:0.004, steps:N}` -> `render.capture_open_level` gave exact, repeatable ages: 12 ms and 40 ms frames of the same muzzle flash that differ correctly, and the 40 ms frame's `imageStats.maxLuminance` falling back to the empty-scene sky value was what proved the effect had ended, not the capture failed. The primitive works; it is the ergonomics that are missing.
  **New interaction worth folding into the ask: the freeze invalidates the documented exposure recipe, and it does so silently.** `render.capture-exposure` and `visual-review.md` both instruct the caller to take one `exposure: {mode:"auto"}` shot, read `viewport.exposure.ev100Equivalent` off it, and pass that number as the pinned `ev100` for the comparison set. Under the freeze, auto-exposure cannot adapt — eye adaptation is driven by world delta time — so that first shot came back `meanLuminance: 0.000154`, `toneLevelsUsed: 5`, `crushed: true`, and reported `ev100Equivalent: 8.906`. Following the documented recipe and pinning 8.9 reproduces the black frame; the scene's real usable value was `ev100: 2`, more than six stops away, found only by bracketing. Both halves are hazardous: the number is wrong, and it is wrong in the one situation the freeze is mandatory for, so every agent that needs the freeze will hit the recipe that the freeze breaks. Whatever form the scoped freeze takes, it should either resolve exposure before stopping time or refuse `{mode:"auto"}` while dilation is below some threshold rather than returning an `ev100Equivalent` measured from an unadapted frame. Evidence: `Saved/Screenshots/OpenLevel/00_stage_auto.png` (auto, crushed), `00c_stage_ev6.png` (pinned 6, `pinnedFrameUsable:false`), `00b_stage_ev2.png` (pinned 2, 83 tone levels, usable). No source read.
- `#4-umg-hud-surface-and-two-dead-workarounds` `OPEN` reporter — Same defect on a **different surface**: UMG in PIE, not Niagara in the editor world. Capturing `/Game/FPS/UI/WBP_HUD`'s hit marker (whole life ~210 ms) is `object.call_function DebugHit` -> `editor.screenshot_window`, two RPCs with the same real-time gap, and it has now defeated three consecutive review rounds — the builder missed it polling `editor.screenshot`, and I missed it twice. Two workarounds that look obvious both fail, and are worth recording so the next agent does not spend a lock slot on them: (1) **`editor.pause` + `editor.step_frame` does not help** — pausing stops the *world* tick but Slate/UMG keep ticking on real time, so the animation runs on through the gap; worse, the paused frames rendered a HUD state the running game never shows (crosshair collapsed while the compass and ammo text stayed painted), i.e. actively misleading evidence (see `E-pause-step-frame-does-not-freeze-umg`). (2) **Time dilation does not help either** — `editor.console_command {command:"slomo 0.06", world:"pie:0"}` is refused outright with `EXEC_FAILED: No exec command or console variable consumed 'slomo 0.06'`, and setting `WorldSettings.TimeDilation = 0.06` via `property.set` *succeeds* (read-back 0.05999) but the marker still missed twice, because `UUserWidget::NativeTick` receives the **Slate** delta, which dilation does not scale. So the UMG surface has no equivalent of `effect.advance_simulation` at all: there is nothing to step. What would fix both surfaces is the atomic form this ticket already asks for — one call that triggers and captures within a single game-thread pass — with a PIE/UMG variant alongside the Niagara one. Cross-referencing `F-pie-capture-fixed-size-with-umg` (same capture path, resolution axis).
