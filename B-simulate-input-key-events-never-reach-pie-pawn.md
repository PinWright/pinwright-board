---
id: B-simulate-input-key-events-never-reach-pie-pawn
title: "editor.simulate_input key_down/key_up report success but never reach a running PIE session — the possessed pawn's input state does not change, so no verb can drive a game under test"
status: OPEN
severity: High
category: bug
tags: [editor, simulate_input, key_down, key_up, pie, enhanced-input, slate-focus, silent-false-success]
encounters: 2
lastSeen: 2026-09-05T21:00:00Z
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

## Fix

**Confirmed TRUE against source.** `EditorCommandHandler.cpp` (pre-fix, `editor.simulate_input`
key branches) built `FKeyEvent(InputKey, FModifierKeysState(), 0, false, 0, 0)` and called
`FSlateApplication::ProcessKeyDownEvent` / `ProcessKeyUpEvent`, discarded the returned handled bool
and set `bSuccess = true`. `FSlateApplication::ProcessKeyDownEvent` (UE 5.8
`SlateApplication.cpp:4961-5072`) routes **only** along `SlateUser->GetFocusPath()` — there is no
game-viewport special case — so unless the PIE game viewport widget is on the keyboard user's focus
path, the event never reaches `FSceneViewport::OnKeyDown` (`SceneViewport.cpp:1266`) →
`UGameViewportClient::InputKey` (`GameViewportClient.cpp:725`) →
`ULocalPlayer::PlayerController->InputKey` → `UPlayerInput`. In a PinWright session focus is on an
editor panel, so keys landed there. The reporter's viewport click did not repair it (it can be
handled by an editor widget without moving focus into the game), and no measurement existed to say
so. Same-cause family as `B-simulate-input-cef-click-noop`, whose `mouse_click` fix left the key
branches alone on purpose.

**Design.** One documented delivery path for game input, in `FDriveGameInput`
(`Handlers/Drive/DriveGameInput.h/.cpp`), built on the existing `FDriveInput` primitives so
`drive.*` and `editor.simulate_input` share one injection layer:

1. **Resolve the destination** — every PIE world context (via the existing
   `PieWorldSelector::GatherPieContexts`), each with its own `UGameViewportClient`
   (`FWorldContext::GameViewport`), local player and player controller. Default pick prefers the
   engine's current game viewport *that has a player controller*; an optional `world` selector
   (`server` / `client` / `client:N` / `pie:N`, the `editor.console_command` grammar) targets a
   specific instance.
2. **Repair focus** — if the game viewport widget is not already on the keyboard user's focus path,
   `FSlateApplication::SetUserFocus` puts it there. Focus is deliberately left on the game (a
   `key_down`/`key_up` pair must share one focus target).
3. **Dispatch the real route** — `FSlateApplication::ProcessKeyDown/UpEvent`, so a focused in-game
   UMG widget (pause menu, CommonUI navigation) sees the key first and the viewport client sees it
   otherwise. This is what a real keystroke does.
4. **Measure at the receiver** — the target `UPlayerInput`'s `FKeyState::EventAccumulator` count for
   that key, sampled before and after. Chosen over `APlayerController::IsInputKeyDown` because
   `bDown` is only assigned during the next `ProcessInputStack`, so it reads `false` for one more
   frame and would report a false negative.
5. **Floor** — only when the focus route provably delivered nothing (no widget consumed it *and* the
   player's input stack never saw it), the key goes straight to `UGameViewportClient::InputKey` with
   the local player's own `FInputDeviceId` (`APlayerController::InputKey` drops events whose device
   maps to another platform user when `bFilterInputByPlatformUser` is on). It cannot double-fire,
   and the response names which route was used.

Rejected alternative: viewport-client delivery *only* (no Slate). It is immune to focus, but a
focused in-game UMG widget never sees the key, which is exactly the second encounter on this ticket
(pause-menu `Escape` then `Down`/`Enter`). Rejected alternative: Slate-only. It leaves the verb dead
whenever focus cannot be moved onto the game — the reported failure mode.

**Response is now measured, never hardcoded:** `route` (`slate` | `viewport_client`), `handled`,
`playerInputRegistered`, `focusRepaired`, `focusedWidget`, `pieInstance`, `kind`, `netMode`, `map`,
`worldPath`, `playerController`, `pawn`. `success` = `handled || playerInputRegistered`; a key
nothing received is `INPUT_FAILED` naming the focused widget. New params: `target`
(`auto` default | `game` | `editor`) and `world`. `target:"game"`, and any call naming a `world`,
refuses with `NO_ACTIVE_SESSION` instead of degrading into an editor keystroke. A missing `key` is
`INVALID_ARGUMENT` and an unknown FKey name is `INVALID_KEY` (both used to be `INPUT_FAILED` prose).

**Files changed**
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveGameInput.h` (new)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveGameInput.cpp` (new)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveInput.h` / `.cpp` —
  `PressKeyReportingHandled` (keyboard counterpart of `ClickAtReportingHandled`); `PressKey`
  delegates to it, contract unchanged for `drive.key`.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp` — key
  branches rewritten; mouse branches untouched.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Drive/TestDriveGameInput.cpp` (new) — 5 tests.
- `Plugins/PinWright/docs/wiki-src/editor.md`, `docs/wiki-src/drive.md`, `docs/rpc-design.md` §1.

**Not compiled, not run** — a separate compile pass follows this change.

**Reviewer verification** (the parts only a live PIE session can prove):

1. `editor.play`, possess a pawn, then `editor.simulate_input {type:"key_down", key:"W"}`. Expect
   `success:true`, `route:"slate"` (or `viewport_client`), `playerInputRegistered:true`, and
   `playerController` / `pawn` naming the possessed pawn. Read the pawn's `GetVelocity` /
   `K2_GetActorLocation` a moment later — it must have moved. Send `key_up` to stop.
2. A key with no binding (e.g. `key:"F9"`): expect `success:true`, `handled:false`,
   `playerInputRegistered:true` — delivered and ignored, distinguishable from never delivered.
3. Pause-menu path: `key_down`/`key_up` on `Escape` with a menu-opening binding, then `Down` /
   `Enter`. The menu must open and navigate (this is the route-through-Slate half).
4. No PIE running: `{type:"key_down", key:"W", target:"game"}` must answer `NO_ACTIVE_SESSION`,
   never `success:true`.
5. Multi-instance PIE (`editor.play {numClients:2, netMode:"listen"}`): `world:"client:1"` and
   `world:"server"` must report different `pieInstance` / `playerController` values.

## History

- `#1-filed` `OPEN` reporter — Hit while reviewing the FPS PLAYER stream's first-person feel, which
  needed held W / LeftShift to photograph walk bob and sprint. Sequence and verbatim responses above,
  all from one 8-minute world-lock slot on `/Game/FPS/Test/T_Player`. The decisive probe is the
  `bWantsSprint` read: a bool written by nothing but the input handler, read directly off the live
  pawn, still `false` with the key held and after a viewport click that Slate said it handled. The
  review that hit this had to fall back to direct function dispatch, and consequently could not
  report measured bob amplitude or a sprint pose at all. No plugin source read.
- `#N-ui-critic-pause-menu-unreachable` `OPEN` reporter — Hit again 2026-09-03T00:30Z on EAContentExamples58, and it is now **blocking a review verdict**, not just an inconvenience. Capturing `/Game/FPS/UI/WBP_PauseMenu` requires opening it the way a player does: `IA_Pause` / `Escape` handled by `BP_HUDManager` (a `PlayerController`). In a live standalone PIE session I sent `editor.simulate_input {type:"key_down"/"key_up", key:"Escape"}`, then `Down`, then `Enter`; every call returned `{"success":true,"message":"Key down: Escape"}`. **No menu ever appeared.** Proof it was input routing and not a dead menu: the same PIE frames show the HUD alive and advancing (compass heading moved 115 -> 309 across the shots, crosshair drawn, ammo painted), and a `mouse_click` at the menu's screen position in the same session *did* report `"was handled by a widget"` — so Slate was live and reachable, only the keyboard path never reached the player controller. Consequence: the one High-severity defect this review had to rule on (pause-menu keyboard/gamepad focus state) could not be exercised by the critic at all, and had to be reported unverified for the second round running. A `key` route that targets the PIE viewport client / player input stack rather than only the focused Slate widget would unblock it.
- `#3-game-input-router` `IN-REVIEW` developer — Confirmed TRUE by reading source only (no editor calls). Root cause is not "no game forwarding" but **wrong routing target plus a discarded handled bool**: `FSlateApplication::ProcessKeyDownEvent` routes solely along the keyboard user's focus path (UE 5.8 `SlateApplication.cpp:4961`), which in a PinWright session is an editor panel, and the handler hardcoded `bSuccess = true`. Fixed by adding `FDriveGameInput` (resolve the PIE `UGameViewportClient` + local player -> put the game viewport on the focus path -> dispatch the real Slate route -> measure the target `UPlayerInput`'s queued-event count -> fall back to `UGameViewportClient::InputKey` only when the focus route provably delivered nothing), and rewiring `editor.simulate_input` key branches onto it with `target` / `world` params and a measured response (`route`, `handled`, `playerInputRegistered`, `focusRepaired`, `focusedWidget`, world/player/pawn identity). Shares the injection layer with `drive.*` via the new `FDriveInput::PressKeyReportingHandled` (keyboard counterpart of the `ClickAtReportingHandled` added by `#2-reword-and-fix` on `B-simulate-input-cef-click-noop`); that ticket's key-branch carve-out is now closed. 5 tests in `Tests/Drive/TestDriveGameInput.cpp` cover the pure destination-selection rule, focus-path containment against a real Slate hierarchy, and the response contract without PIE (`target:"game"` with no session must be `NO_ACTIVE_SESSION`, bad key/target refused by code, editor route reports `handled`). Not compiled and not executed here — a separate compile pass follows; the live-PIE half is the reviewer checklist in the Fix section. Details, rejected designs and file list: `## Fix` above.

- `#4-verified-in-fps-build` `IN-REVIEW` reporter - **Fixed. Verified live in the FPS build, 2026-09-05 17:52Z, UE 5.8, shared editor port 27145, plugin at origin/master after the wave 4-9 rebuild.** Call: `editor.simulate_input {type:"key_down", key:"LeftMouseButton", target:"game"}` into a running standalone PIE session on `/Game/FPS/Test/T_Weapons`. Response now carries `route:"slate"`, `handled:true`, `playerInputRegistered:true`, `focusRepaired:true`, `focusInGameViewport:true`, `focusedWidget:"SViewport"`, and names `worldPath /Game/FPS/Test/UEDPIE_0_T_Weapons.T_Weapons`, `playerController PlayerController_0` and `pawn BP_WeaponTestPawn_C_0`.
  **Not taken on the response's word** - the ticket's actual claim was that the possessed pawn's state does not change, so that is what was measured. The pawn's equipped `BP_Weapon_AR_C_2` went from `AmmoInMag 30 / ShotCounter 0` to `AmmoInMag 0 / ShotCounter 30` while the key was held (auto fire at 620 rpm), and the placed `BP_EnemyCharacter_C_0` dropped from full health to `Health 22.0` from those hits. A second burst after `key_up` / `key_down` took the counter to 46. Gameplay observes the key: movement, weapon, recoil and ADS reviews all have a route now.
  Ancillary, same session: the player controller's control rotation moved from the pitch -0.19 deg set before the burst to +9.15 deg after 16 shots, so recoil applied through the real input path rather than through a scripted call.
- `#N-returned` `OPEN` reporter — **Returning this to OPEN: the fix works for one Enhanced Input action and not for three others, and the response cannot tell them apart.** UE 5.8 / EAContentExamples58, plugin at `origin/master` post-waves-4-9, one PIE session on `/Game/FPS/Test/T_Player` owned by this stream.

  Every call below returned `route: "slate"`, `handled: true`, `playerInputRegistered: true`, `focusInGameViewport: true`, `focusedWidget: "SViewport"`, and named the correct `PlayerController_0` / `BP_FPSCharacter_C_0`:

  | key | action | trigger | game-side result |
  |---|---|---|---|
  | `LeftShift` | `IA_Sprint` | Started | **worked** — `bWantsSprint` went True |
  | `W` | `IA_Move` | Triggered (Axis2D) | nothing — `speed` stayed `0.0`, sampled twice seconds apart |
  | `R` | `IA_Reload` | Started | nothing — `bIsReloading` False, no montage |
  | `LeftMouseButton` | `IA_Fire` | Started | nothing — `AmmoInMag` stayed 30, `KickAlpha` 0 |

  `mouse_click` at the viewport centre was also tried for `IA_Fire`: `"Mouse click at (958, 540) was handled by a widget"` — and again nothing fired.

  **The game side is proven healthy independently**, which is what makes this a delivery problem rather than a project bug: calling the weapon's `Reload` directly on the same actor in the same session gave `bIsReloading: True` and `Montage_IsPlaying: True` on the arms anim instance immediately. So the mapping context, the action assets, the handlers, the dispatchers and the montage all work; only the synthetic key fails to become an Enhanced Input trigger for three of the four actions.

  Two things make this expensive to a caller. First, `playerInputRegistered: true` is reported identically for the action that fired and the three that did not, so there is no in-band way to tell a delivered key from a swallowed one — the only signal is a game-side variable the caller has to know to read. Second, it is **intermittent across sessions**: in an earlier slot the same `W` key did move the pawn several metres, and in this one it produced zero velocity, so a caller cannot even learn a stable workaround.

  Suggest the response distinguish "the key was injected" from "an Enhanced Input action triggered" — the subsystem knows which actions fired on that tick, and reporting them by name would make this self-diagnosing. Workaround in use: call the gameplay method directly (`Reload`) or set the driving variable, and label such captures as not-input-driven.
