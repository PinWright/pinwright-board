---
id: B-editor-background-throttle-stalls-rpcs
title: "PinWright never disables editor background throttling, so an unfocused editor runs at 3 FPS during agent sessions and every RPC and multi-frame job waits on 333 ms frames"
status: IN-REVIEW
severity: High
category: bug
tags: [performance, throttling, editor, background, request-pump, jobs, pcg, benchmark, capture, pie, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# Unfocused editor throttles to 3 FPS during agent sessions

An agent drives the editor from another window, so the editor is almost always in the background.
The engine then caps the editor at 3 FPS and stops viewport rendering, and PinWright does nothing to
opt out. Everything PinWright does on the game thread is paced by that frame rate.

Engine side (UE 5.8 lines; the same logic exists on 5.3 and 5.5):

- `UEditorPerformanceSettings::bThrottleCPUWhenNotForeground` defaults to `true`
  (`Engine/Source/Editor/UnrealEd/Private/EditorPerformanceSettings.cpp:21`).
- `UEditorEngine::ShouldThrottleCPUUsage` (`Engine/Source/Editor/UnrealEd/Private/EditorEngine.cpp:5305-5350`)
  returns true when the app is not foreground and has no focus and the setting is on, or when all
  windows are hidden (minimized) regardless of the setting. It first returns false if any bound
  `ShouldDisableCPUThrottlingDelegates` entry returns true (`:5318-5324`).
- `UEditorEngine::GetMaxTickRate` then returns `MaxTickRate = 3.0f` (`:2563`).
- Separately, `:1805-1815` adds a "Background Process" realtime override that turns off every editor
  viewport while `!FApp::HasFocus()` and the setting is on. That check does not consult the
  delegates.
- Commandlets, `-unattended` and benchmarking skip the throttle (`:5308-5314`); a normal windowed
  editor launched for an agent session does not.

PinWright side (plugin HEAD `71c91649`): no source file references
`ShouldDisableCPUThrottlingDelegates` or `bThrottleCPUWhenNotForeground`. The request pump is a
core-ticker callback (`Source/PinWright/Private/PinWrightSubsystem.cpp:215-218`) that drains
pending requests once per pass (`UPinWrightSubsystem::Tick`), and the core ticker advances once per
engine frame. At 3 FPS each queued RPC can wait up to about 333 ms before dispatch, and anything that
needs several frames multiplies that: PIE start and stop, captures and screenshots, PCG generation,
asset compilation waits, benchmarks.

Observed on this board already:

- `B-pcg-generate-deadlocks-game-thread`: completion is a multi-frame wait that "an unfocused /
  throttled editor (the normal state in an agent session) stretches ... into minutes", past the
  300 s transport timeout.
- `B-performance-run-benchmark-measures-nothing` (`#2-verified-in-fps-build`): `avgFps 2.99999`, every frame within
  4 microseconds of 333.3333 ms, reported as a scene cost.
- `B-set-window-state-cannot-restore-minimized`: 150 frames at 333.33 ms on a minimized editor,
  which turned into an end-user "3 FPS in the forest" report.

Workaround: set Editor Preferences > Performance > "Use Less CPU when in Background" off by hand
(it does not cover the minimized case), or keep the editor focused.

Engines: all supported (5.3 to 5.8).

**Fix:** register a delegate on `GEditor->ShouldDisableCPUThrottlingDelegates` at subsystem startup
(remove it on shutdown) that returns true while PinWright is in use: an RPC was dispatched within a
recent window, or any job is running. `Source/PinWright/Private/State/ClientActivity.cpp` already
stamps every dispatch; it needs a "most recent dispatch by any client" accessor next to
`GetMostRecentOtherClient`, and `FJobRegistry` needs a running-count query. Because the viewport
realtime override ignores the delegates, verbs that render through an editor viewport also need the
override lifted for their duration (or the setting toggled while active). Add a
`UPinWrightProjectSettings` opt-out for users who want the engine default. Report the throttle
state in `performance.run_benchmark` output when it is on, so a clamped measurement is labelled.

**Related:** `B-pcg-generate-deadlocks-game-thread`, `B-performance-run-benchmark-measures-nothing`,
`B-set-window-state-cannot-restore-minimized`.

## History
- `#1-background-throttle` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified at plugin HEAD `71c91649`: no PinWright code binds `ShouldDisableCPUThrottlingDelegates`; engine caps the unfocused editor at 3 FPS (`EditorEngine.cpp:2563`, setting default `EditorPerformanceSettings.cpp:21`) and disables viewports (`EditorEngine.cpp:1807`); the request pump is a core-ticker pass per frame (`PinWrightSubsystem.cpp:215-218`). Consistent with the 333.33 ms frames recorded in three existing tickets. Board has no performance category; filed as a bug tagged `performance`. Severity High: a soft blocker (Medium impact) on the path every agent session takes, bumped up one for reach.
- `#2-throttle-guard` `IN-REVIEW` developer - Added `State/BackgroundThrottleGuard.{h,cpp}`, owned by `UPinWrightSubsystem` (registered in `Initialize`, updated every `Tick`, unregistered in `Deinitialize`). While active (an RPC dispatched within `EditorQuitPolicy::InUseWindowSeconds` = 300 s via new `ClientActivity::GetSecondsSinceLastDispatch`, or `FJobRegistry::NumRunning() > 0`) it answers true from a `GEditor->ShouldDisableCPUThrottlingDelegates` entry and clears `UEditorPerformanceSettings::bThrottleCPUWhenNotForeground` in memory (the viewport "Background Process" override reads it directly); the user's value is restored on lapse and on shutdown, never saved (an Editor Preferences save of that section during the hold is re-saved with the user's value; a user edit of the flag mid-hold is adopted). Opt-out: `UPinWrightProjectSettings::bDisableBackgroundThrottleWhileAgentActive` (default true). Minimized limit documented in `docs/wiki-src/performance-profiling.md`: the tick cap lifts, but `UEditorEngine::Tick` still skips viewport redraw with all windows hidden. APIs identical 5.3-5.8, no guards. Tests `PinWright.state.background_throttle.*` (4). Not done: labelling throttle state in `performance.run_benchmark` output (outside the approved plan's task 2 scope). Not compiled or run yet (wave build pending).
- `#3-benchmark-throttle-label` `IN-REVIEW` developer - Remaining scope done: `performance.run_benchmark` now samples throttle state on every frame it measures and publishes `backgroundThrottle {throttledFrames, backgroundFrames, guardHeldFrames, throttleCPUWhenNotForegroundFrames}` (engine `GEditor->ShouldThrottleCPUUsage()`; `!IsThisApplicationForeground() && !FApp::HasFocus()`; new `UPinWrightSubsystem::IsHoldingBackgroundThrottle()`; effective `bThrottleCPUWhenNotForeground`), plus a `warnings` entry when `throttledFrames > 0` (`Handlers/Debug/PerformanceHandler.cpp`). Test `PinWright.performance.run_benchmark.ReportsBackgroundThrottleState` (counts bounded by frameCount, a held frame is never throttled, and with the project setting on the running benchmark job makes the guard hold). Documented in `docs/wiki-src/performance.md` and `performance-profiling.md`. Not compiled or run yet (wave build pending).
