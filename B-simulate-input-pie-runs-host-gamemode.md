---
id: B-simulate-input-pie-runs-host-gamemode
title: "PinWright.editor.simulate_input.PiePlayerInputDelivery starts PIE under the host project's default GameMode, so host content runs inside a plugin test and crashed the suite"
status: IN-REVIEW
severity: High
category: bug
tags: [tests, pie, simulate-input, host-dependence, suite-crash, gap-analysis-2026-09-28]
encounters: 3
costly: 1
lastSeen: 2026-09-29T16:54:14Z
---

# PiePlayerInputDelivery runs host content inside a plugin test

`PinWright.editor.simulate_input.PiePlayerInputDelivery`
(`Source/PinWright/Private/Tests/Drive/TestDriveGameInput.cpp:396`) starts its owned PIE session
with the engine's `FStartPIECommand(false)` on the suite's blank untitled world and no GameMode
override. The PIE world therefore spawns whatever the host project's default GameMode and default
pawn are. On a Lyra-based host that means the experience manager loads the host's experience and
spawns the host's pawn Blueprint, so host gameplay code runs inside a plugin test.

Observed on the gap-quick-wins verification run (host log `Saved/Logs/pw_gapwave_groups.log`,
lines 14035-14110; crash report `Saved/Crashes/UECC-Windows-07F56CD54DB99C343E61C7B6FD94FFF1_0000`):
the editor died with

```
Assertion failed: !FUObjectThreadContext::Get().IsRoutingPostLoad [ScriptCore.cpp:2061]
Cannot call UnrealScript (<host pawn BP> ... Function <host BP>:ReceiveAsyncPhysicsTick) while PostLoading objects
```

Callstack (innermost first): `UObject::ProcessEvent` <- `AActor::ReceiveAsyncPhysicsTick` <- host
game module <- `FAsyncPhysicsTickCallback::OnPreSimulate_Internal` <- Chaos `GTPreSimCallbacks`
pumped on the game thread by `FNamedTaskThread::ProcessTasksUntilIdle` <- `FRenderCommandFence::Wait`
<- `UTexture2D::UpdateResourceWithParams` <- `UTexture::PostLoad` <- `FlushAsyncLoading` <-
`LoadPackage` <- `FSoftObjectPath::TryLoad` <- `ULyraExperienceManagerComponent::SetCurrentExperience`
<- `ALyraGameMode::OnMatchAssignmentGiven` <- `FTimerManager::Tick` <- `UWorld::Tick`. No PinWright
frame is on the stack: the crash is host content interacting with the engine, but it is reachable
only because the plugin test PIEs under the host's GameMode. The queue had not drained, so every
test after it (the rest of `editor.*`, `gameplay_tags`, `infra`, `performance`, `state`, `system`,
`transport`) went unmeasured.

Plugin tests must not depend on host content (plugin `CLAUDE.md`, "This plugin is the deliverable,
and it is general-purpose"; the suite-start blank world exists for the same reason).

**Workaround:** exclude the test from scoped runs (RunTests has no exclusion syntax; list the
`editor.*` subgroups explicitly and the other `simulate_input` tests with `^...$`).
**Fix:** start the owned PIE with a neutral GameMode override for the test world (e.g. set
`AWorldSettings::DefaultGameMode` on the transient world to `AGameModeBase` with a plain
`ADefaultPawn`, or start PIE through `FRequestPlaySessionParams` with a GameModeOverride), so the
test measures input delivery and never spawns host pawns or loads host experiences.

## History
- `#1-suite-crash-host-gamemode` `OPEN` reporter — Filed from the gap-quick-wins verification run: PIE under the host's Lyra GameMode loaded the host experience and the host pawn's `ReceiveAsyncPhysicsTick` asserted during a nested PostLoad, crashing the scoped suite (`pw_gapwave_groups.log`, crash `UECC-Windows-07F56CD54DB99C343E61C7B6FD94FFF1_0000`). Cost: one scoped suite run lost its second half.
- `#2-assertions-fail-under-host-frontend` `OPEN` reporter — Additional evidence once the StateTree leak was fixed and the test stopped skipping (`Saved/PinWright/test-runs/b99b93ab15424456b7000b4b6af1de26/automation.log:8712-8900`): the owned PIE ran `Game class is 'B_LyraGameMode_C'` with experience `B_LyraFrontEnd_Experience`; CommonUI switched to `ECommonInputMode::Menu` and focused a front-end button, so the UI consumed the keys. `key_down is delivered to game` and the following delivery/route/PlayerInput assertions failed at `TestDriveGameInput.cpp(321/327/331/335/358)`.
- `#3-host-neutral-owned-pie` `IN-REVIEW` developer — New `Tests/Drive/HostNeutralPie.h` defines `FStartHostNeutralPieCommand`: it builds the same `FRequestPlaySessionParams` as the engine's `FStartPIECommand` and adds `GameModeOverride = AGameModeBase::StaticClass()`. The engine writes that onto the PIE world's `WorldSettings->DefaultGameMode` before the game instance starts (`PlayLevel.cpp` CreateInnerProcessPIEGameInstance), so the override holds on any editor map, including a host map without `aa_suite_start`, and never touches the editor world; nothing needs restoring. PIE then runs a plain APlayerController, ADefaultPawn and AHUD: no host GameMode and no experience-driven UI. Level-placed actors of a host map and the host GameInstance class still run. Both plugin tests that start PIE directly (`editor.simulate_input.PiePlayerInputDelivery`, `drive.observe.ScreenshotIncludesUmgAndMarksAtSurfaceLocalCoords`) now use it; no other test starts an owned PIE. Assertions unchanged. `-SingleFile` compile: clean.
- `#4-host-plugin-pie-start-error` `IN-REVIEW` developer — The full offscreen suite with errors unmasked (`Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log:28524-28639`) failed `PiePlayerInputDelivery` on `LogPixelStreaming2RTC: Error: Cannot set target window - GEngine is not valid.`. The host's PixelStreaming2 editor module creates a PIE streamer on `PostPIEStarted` while `PixelStreaming2.Editor.AutoStreamPIE` is on (the default). The streamer's input handler logs this once per PIE start because GEngine is not a `UGameEngine` (`RTCInputHandler.cpp:184`); the GameMode override cannot prevent it. `drive.observe.ScreenshotIncludesUmg...` logged the same line at 27279 but passed only because it set a blanket `bSuppressLogErrors = true` in `RunTest`. Fix: `Tests/Drive/HostNeutralPie.h` `FStartHostNeutralPieCommand` now takes the test and, when the `PixelStreaming2Editor` module is loaded and that CVar is on, declares exactly that message as one expected error before requesting PIE. Both PIE tests pass `this`, and drive.observe's blanket suppression is removed, so both tests now measure identically. `-SingleFile` compile of both tests: clean.
- `#5-pixelstreaming-gate-mirrors-plugin` `IN-REVIEW` developer — The full HEADLESS suite (`Saved/PinWright/test-runs/af9a1781f41e48038b90221f51d589d7/automation.log` ~27515) failed `PiePlayerInputDelivery` on the #4 expectation: found 0 times. Under `-NullRHI` PixelStreaming2 never logs the line: that log has no PixelStreaming2 line at all, not even the one at editor startup. The cause is the plugin's own gate. `FPixelStreaming2EditorModule` binds its `PostPIEStarted` hook only in `InitEditorStreaming`, which runs from PixelStreaming2RTC's `OnReady`. `FPixelStreaming2RTCModule::StartupModule` returns before it can ever become ready unless the dynamic RHI is D3D11, D3D12, Vulkan or Metal (OpenGL on Android only). `IsPixelStreamingPieStreamerLive()` in `Tests/Drive/HostNeutralPie.h` now requires the same RHI set, plus the editor module being loaded and `PixelStreaming2.Editor.AutoStreamPIE` on, before declaring the error. It is not a blanket headless skip, and there is no link dependency on the plugin. drive.observe goes through the same helper; it skips with `reason=null-rhi` before starting PIE under NullRHI anyway. `-SingleFile` compile of both PIE tests: clean.
- `#6-committed-verified-headless-offscreen` `IN-REVIEW` developer — Committed in PinWright `0140e986` ("Fix two tests that failed only under headless -NullRHI runs"). Verified on UE 5.8: build `Result: Succeeded`; targeted runs (`aa_suite_start` + `editor.simulate_input` + `drive.observe` + `Niagara.DumpCompile` + `zz_suite_end`) passed 15/15 headless and 15/15 offscreen, including `PiePlayerInputDelivery` under both.
