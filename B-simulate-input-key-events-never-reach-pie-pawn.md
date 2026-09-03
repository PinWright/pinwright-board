---
id: B-simulate-input-key-events-never-reach-pie-pawn
title: "editor.simulate_input key_down/key_up report success but never reach a running PIE session — the possessed pawn's input state does not change, so no verb can drive a game under test"
status: OPEN
severity: High
category: bug
tags: [editor, simulate_input, key_down, key_up, pie, enhanced-input, slate-focus, silent-false-success]
encounters: 1
lastSeen: 2026-09-02T21:56:00Z
---

# Simulated key events never reach the PIE pawn

`editor.simulate_input {type:"key_down"}` returns `{success:true, message:"Key down: W"}` while a
PIE session is running and possessing a pawn, but nothing in the game observes the key. Every
gameplay review that needs the game *driven* — movement feel, weapon bob, sprint, ADS, recoil — has
no route at all, because this is the only input-injection verb.

## What I called

FPS PLAYER-critic slot, 2026-09-02T21:49–21:57Z, UE 5.8 / `EAContentExamples58`, standalone PIE on
`/Game/FPS/Test/T_Player`, `BP_FPSCharacter_C_0` possessed by `PlayerController_0`
(`editor.status` -> `inPie:true`, `playerControllerPath` populated).

```
editor.simulate_input {type:"key_down", key:"W"}
-> {"success":true,"type":"key_down","message":"Key down: W"}
object.call_function {..BP_FPSCharacter_C_0, function:"GetVelocity"}   -> [0,0,0]
object.call_function {..BP_FPSCharacter_C_0, function:"K2_GetActorLocation"} -> [0,0,92.15]
   (unchanged across two reads spanning a screenshot round-trip, ~1 s of live PIE)
```

Suspecting Slate focus, I clicked into the viewport first — which reported that a widget handled it —
and repeated:

```
editor.simulate_input {type:"mouse_click", x:680, y:500, button:"left"}
-> {"success":true,"message":"Mouse click at (680.000000, 500.000000) was handled by a widget"}
editor.simulate_input {type:"key_down", key:"W"}   -> success
K2_GetActorLocation -> [0,0,92.15]                 (still not moved)
```

A key that sets a plain bool, read straight off the pawn, is the cleanest possible probe and it also
fails:

```
editor.simulate_input {type:"key_down", key:"LeftShift"}  -> success
property.get {..BP_FPSCharacter_C_0, propertyName:"bWantsSprint"} -> false
```

`bWantsSprint` is set by nothing but the sprint input handler, so `false` here means the action never
fired — it is not a movement-component or collision question.

## What I expected

The key to reach the focused game viewport and be consumed by the PIE player's input stack, the way
a real keystroke is, so a critic can hold W and photograph the result. Failing that, an error saying
the event was not routed — not `{success:true}`.

## Why this is not the game's fault

The same pawn responds correctly to direct function dispatch in the same session:
`object.call_function` of `LaunchCharacter`, `Jump`, `Crouch` and `HandleMove` all take effect
(measured: actor X moved 34 -> 465.9 after `LaunchCharacter`; `Crouch` dropped the camera 164.15 ->
124.15). So the pawn, its movement component and its Blueprint functions are live and reachable; only
the injected key is not. The Blueprint's Enhanced Input nodes may or may not be bound — this ticket
cannot tell, and that is exactly the problem: **a silent-success input verb makes a game defect and a
tool defect indistinguishable.**

## Relationship to the existing ticket

`B-simulate-input-cef-click-noop` (IN-REVIEW, High) covers the **`mouse_click`** branch: nullptr
platform window, no effecting button, hardcoded `bSuccess = true`. This is the **`key_down` /
`key_up`** branch and a different destination (a PIE game viewport rather than an editor widget), so
it is filed separately; if the root cause turns out to be the same hardcoded success plus a bad
routing target, merge them. The hardcoded-success half is certainly shared: both branches answer
`success:true` without consulting whether anything handled the event.

## Workaround

None for input as such. To exercise a pawn's states I drove them mechanically instead — direct
`object.call_function` on the pawn's own functions, and `actor.set_blueprint_variables` to force
state flags (`bIsADS`). That reaches the state but **not** the input path, so it cannot answer "does
the player's binding work", which is often the question being asked.

## Suggested fix

Return the handled-bool from `FSlateApplication::ProcessKeyDownEvent` rather than hardcoding
`true`, and route to the game viewport widget (`UGameViewportClient`'s Slate viewport) when a PIE
session is active — or add an explicit `target: "pie" | "editor"` parameter — so the caller can tell
"delivered and ignored" from "never delivered".

## History

- `#1-filed` `OPEN` reporter — Hit while reviewing the FPS PLAYER stream's first-person feel, which
  needed held W / LeftShift to photograph walk bob and sprint. Sequence and verbatim responses above,
  all from one 8-minute world-lock slot on `/Game/FPS/Test/T_Player`. The decisive probe is the
  `bWantsSprint` read: a bool written by nothing but the input handler, read directly off the live
  pawn, still `false` with the key held and after a viewport click that Slate said it handled. The
  review that hit this had to fall back to direct function dispatch, and consequently could not
  report measured bob amplitude or a sprint pose at all. No plugin source read.
- `#N-ui-critic-pause-menu-unreachable` `OPEN` reporter — Hit again 2026-09-03T00:30Z on EAContentExamples58, and it is now **blocking a review verdict**, not just an inconvenience. Capturing `/Game/FPS/UI/WBP_PauseMenu` requires opening it the way a player does: `IA_Pause` / `Escape` handled by `BP_HUDManager` (a `PlayerController`). In a live standalone PIE session I sent `editor.simulate_input {type:"key_down"/"key_up", key:"Escape"}`, then `Down`, then `Enter`; every call returned `{"success":true,"message":"Key down: Escape"}`. **No menu ever appeared.** Proof it was input routing and not a dead menu: the same PIE frames show the HUD alive and advancing (compass heading moved 115 -> 309 across the shots, crosshair drawn, ammo painted), and a `mouse_click` at the menu's screen position in the same session *did* report `"was handled by a widget"` — so Slate was live and reachable, only the keyboard path never reached the player controller. Consequence: the one High-severity defect this review had to rule on (pause-menu keyboard/gamepad focus state) could not be exercised by the critic at all, and had to be reported unverified for the second round running. A `key` route that targets the PIE viewport client / player input stack rather than only the focused Slate widget would unblock it.
