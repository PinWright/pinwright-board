---
id: F-observe-standalone-game-process
title: "No way to observe a standalone -game process: editor.launch_standalone spawns it, then drive.observe / screenshots / DOM export all stop at the editor boundary"
status: OPEN
severity: Medium
category: feature
tags: [drive, observe, screenshot, standalone, launch-standalone, cef, webui, verification]
encounters: 1
costly: 1
lastSeen: 2026-09-02T00:00:00Z
---

# No way to observe a standalone `-game` process

`F-editor-launch-standalone` (DONE) closed the **launch** half: `editor.launch_standalone`
spawns `UnrealEditor.exe <project> -game …` and returns PIDs. Nothing closes the
**observation** half. Every inspection verb — `drive.observe`, the web/DOM export,
`ui.screenshot`, `render.*` captures — is served by the in-editor RPC gateway
(`UPinWrightSubsystem`, HTTP on the per-project port), which only exists inside the
editor process. The spawned `-game` instance binds nothing, so the moment a
behaviour is `-game`-only it is unverifiable through PinWright and the agent falls
back to log grepping.

## Where this bit (PDS, 2026-09-02, at least four agents)

The project's acceptance gate runs as
`UnrealEditor.exe <project> -game -RunAcceptanceTests`
(`docs/acceptance-testing.md`; a Development `-game` run is recorded there for
2026-09-02). Its operator-facing surface is a CEF overlay,
`Content/WebUI/AcceptanceHUD/index.html`, hosted by `UAppAcceptanceWebHost` and
rendered **only in that process**. Consequences observed in one session:

- `drive.observe` and the DOM export were unavailable for the page under test —
  they reach the editor, which is not running the overlay.
- A desktop-level screen capture "got the desktop only".
- The `ui-edit` skill's browser route needs the editor's HTTP server on the
  project port, which a `-game` run does not expose.
- Verification fell back to log analysis plus one manual photograph of the screen.

PIE is not a substitute here: the overlay's subsystems are gated on
`FParse::Param(FCommandLine::Get(), TEXT("RunAcceptanceTests"))`, and a command
line cannot be delivered to an in-editor PIE session the way it can to a spawned
process. The same wall applies to any `-game`-only UI, HUD or CEF panel.

## What would close it

In rough order of cost, any one of these makes the class of task possible:

1. **A minimal observe/capture surface inside the runtime module.** PinWright
   already ships runtime-side modules; binding the RPC transport in a `-game`
   build (opt-in via a launch flag, on a port the spawner reports back) would let
   `drive.observe` / screenshot verbs address the spawned instance the same way
   they address the editor. `editor.launch_standalone` already returns the PID and
   command line and is the natural place to pass the flag and echo the port.
2. **Client-side addressing.** If the transport can be hosted there, the missing
   half is a way to say *which* instance a call targets — today the port is derived
   per project, not per process.
3. **A documented pattern, if neither is in scope.** Even a wiki page stating that
   `-game` instances are unobservable, and naming the sanctioned fallbacks
   (`-ExecCmds` screenshot commands, `HighResShot`, log-driven assertions), would
   stop the next agent from spending a session discovering it — that is what
   happened here.

Not to be confused with `F-editor-launch-standalone` (spawning: DONE) or with the
PIE-side verbs, which work fine because PIE runs inside the editor process.

## History
- `#1-feature-request` `OPEN` reporter — Filed from a PDS session (2026-09-02) in which at least four agents needed to verify the acceptance overlay (`Content/WebUI/AcceptanceHUD`, hosted by `UAppAcceptanceWebHost`) that renders only in the `UnrealEditor.exe <project> -game -RunAcceptanceTests` process. PinWright's transport lives in the editor process only, so `drive.observe`, DOM export and every screenshot verb stopped at the process boundary: a desktop capture "got the desktop only", the `ui-edit` browser route needs the editor's HTTP server on the project port which the `-game` run does not expose, and verification degraded to log analysis plus a manual photograph. PIE is not a substitute — the overlay's subsystems gate on `-RunAcceptanceTests`, which cannot be delivered to an in-editor PIE session. `F-editor-launch-standalone` (DONE) covers spawning only; nothing on the board covers observing the spawned process. severity rationale: impact=High-or-Medium band — a hard blocker with no in-tool workaround, so verifying any `-game`-only UI is impossible through PinWright × reach=rare path (most work targets the editor or PIE, both already observable), bump down -> Medium. Asked for, in cost order: bind the RPC transport in a `-game` build behind a launch flag with the port echoed by `editor.launch_standalone`; per-process addressing for the observe verbs; or, failing both, a wiki page stating the limitation and naming the sanctioned fallbacks.
- `#2-still-open-at-upstream-head-plus-premise-correction` `OPEN` reporter — Re-verified against plugin HEAD `347826a6` after the 398-commit pull from `b16f0f2b`. **Still unimplemented**; status stays `OPEN`/`Medium`, `encounters` unchanged (source re-read). Nothing upstream binds a transport outside the editor: `PinWright.uplugin` at HEAD declares **seven modules, all `"Type": "Editor"`** (`PinWrightRecorder`, `PinWright`, `PinWrightGeometry`, `PinWrightPCG`, `PinWrightChooser`, `PinWrightPoseSearch`, `PinWrightCommonUI`), and `Source/` holds no runtime module. `editor.launch_standalone` still exists and is still spawn-only (`Source/PinWright/Private/Handlers/Editor/EditorLaunchHandler.cpp:54`); the observe verbs are still editor-side handlers (`Source/PinWright/Private/Handlers/Drive/DriveObserveHandler.cpp`). No wiki page under `docs/wiki-src/` states the `-game` observation limitation or names the sanctioned fallbacks, so option 3 is not done either. **Correction to `#1`:** the sentence "PinWright already ships runtime-side modules" is **false** and was equally false before the pull — the module-type census above is byte-identical between `b16f0f2b` and `347826a6` (7x `"Type": "Editor"` in both). Option 1 is therefore more expensive than `#1` implies: it requires creating a Runtime module and gating the transport in it, not just flipping a switch on an existing one. The rest of `#1` stands unchanged.
