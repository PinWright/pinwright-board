---
id: E-editor-play-gamemode-override
title: "editor.play has no optional gameModeOverride, so its only happy-path test must skip in every unattended run and agents cannot PIE a map under another GameMode without editing World Settings"
status: OPEN
severity: Low
category: ergonomic
tags: [editor-play, pie, gamemode, tests, unattended, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
rice: [1, 1, 1, 1]
priority: 8
---

# editor.play: optional GameMode override

`editor.play` (`Handlers/Editor/PIEHandler.cpp:252`) starts PIE with whatever GameMode the level and
project resolve; it has no parameter to override it. For an agent testing its own game that default is
correct and must stay the default. Two consequences of having no opt-in override:

1. **The verb's PIE-start path has no unattended coverage.** `PinWright.editor.play.ValidParamsNoCrash`
   (`Tests/EditorOps/TestEditorHandlers.cpp:705-722`) is the only test that calls the production
   `editor.play` and starts PIE; it skips under `-unattended`. The lifecycle tests in
   `Tests/EditorOps/TestEditorPieLifecycle.cpp` register a stand-in `editor.play` handler, and the
   network-emulation tests are forbidden to start PIE. `B-transient-test-blueprints-compiled-every-pie`
   `#2` keeps this skip deliberately because it "also guards host-GameMode PIE for the production verb":
   on a Lyra host, PIE under the host GameMode loads host experiences and crashed a suite
   (`B-simulate-input-pie-runs-host-gamemode`). That ticket's fix (`Tests/Drive/HostNeutralPie.h`,
   `FRequestPlaySessionParams::GameModeOverride = AGameModeBase`) works for tests that start PIE
   themselves, but a test of `editor.play` must go through the verb.
2. **Agents** that want to PIE a map under a different GameMode (a test harness mode, a menu-less mode)
   must edit and later restore the level's World Settings.

**Impact:** a coverage gap on an every-session verb and a small agent workflow gap; both are friction.
**Fix:** optional `gameModeOverride` (class path, resolved with `ResolveUClass`, must be a
`AGameModeBase` subclass, typed error otherwise) copied into `FRequestPlaySessionParams::GameModeOverride`
(present on UE 5.3-5.8), echoed in the response and in `editor.pie_status`. Omitted means today's
behaviour. Then `ValidParamsNoCrash` passes `/Script/Engine.GameModeBase` and drops the unattended
skip once `B-transient-test-blueprints-compiled-every-pie` is DONE.

## History
- `#1-no-gamemode-override` `OPEN` reporter — Filed from today's verification; judged worth tracking because it is the only way to give the production `editor.play` start path unattended coverage without running host content. Verified: no GameMode handling in `PIEHandler.cpp`; the skip and its stated reasons at the lines above.
