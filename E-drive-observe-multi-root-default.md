---
id: E-drive-observe-multi-root-default
title: "drive.observe on the game surface fails AMBIGUOUS_LIVE_ROOT whenever two UMG roots are on the viewport (PDS: JoinLeaveHUDNativeOverlay + W_OverallUILayout_C_0), so every observe needs instance_name"
status: DONE
severity: Medium
category: ergonomic
tags: [drive, drive.observe, live-root, ambiguous-live-root, instance-name, pie, game-surface]
encounters: 4
lastSeen: 2026-09-29T13:05:00Z
---

# A whole-surface observe should not need a root selector

`drive.observe` on the game surface (PIE) without `instance_name` returned:

```
[AMBIGUOUS_LIVE_ROOT] Multiple live UMG root candidates matched: ... (JoinLeaveHUDNativeOverlay), ... (W_OverallUILayout_C_0).
Pass instance_name (the backing widget name, e.g. from ui.create_hud) or root_index to select one.
```

In PDS this is every PIE session, not an edge case: `UAppJoinLeaveHUDSubsystem`
(`Plugins/App/Source/App/UI/JoinLeave/AppJoinLeaveHUDSubsystem.cpp:100`) adds a `JoinLeaveHUDNativeOverlay` widget to
the viewport for the whole game instance, next to the main `W_OverallUILayout_C_0`. Any project with a HUD plus an
always-on overlay hits the same.

The drive resolver reuses the `widget.describe` root selector unchanged
(`Handlers/Drive/DriveLiveResolver.cpp:570-592` -> `FLiveUiSnapshotService::SelectRootCandidate`,
`Handlers/UI/LiveUiSnapshot.cpp:772-787`). That rule fits `widget.describe`, which dumps one widget tree. It fits
`drive.observe` less well: the caller asked "what is on screen", and the answer spans all roots. The error is
clear and the retry is one call, but the retry is needed on every observe, and a caller who picks the wrong root
silently misses controls in the other one (for example a join/leave toast over the menu).

**Workaround:** pass `instance_name: "W_OverallUILayout"` (or the overlay name) on every call.

**Fix (proposed):** when no selector is given, `drive.observe` (and the other drive verbs that resolve handles
on the game surface) walks every live root in viewport z-order and tags each element with its root name, instead
of failing. Keep `instance_name` / `root_index` to narrow. Handle uniqueness already spans the walked set
(`AssignDriveHandles`), so merging roots should not need new handle rules.

## History
- `#1-ambiguous-on-every-pie-observe` `OPEN` reporter - UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`, PIE in PDS map-editor mode. Every game-surface observe needed `instance_name`. Severity: Low impact (one retry, error names the parameter), bumped one level for reach since drive.observe runs in nearly every PDS PIE session. Related: `F-widget-describe-live-root-disambiguation` (added the selector this ticket wants to be optional for drive).
- `#2-map-editor-pie-again` `OPEN` reporter - Seen again in the PDS map editor (PIE, `App.Launch mapeditor`): a bare `drive.observe {screenshot:false, interactables_only:true, max_elements:4}` returned `AMBIGUOUS_LIVE_ROOT` naming `JoinLeaveHUDNativeOverlay, W_OverallUILayout_C_0`; retry with `instance_name:"W_OverallUILayout"` worked. Only needed the geometry of one HUD button to map viewport pixels to desktop coordinates for an XTEST gizmo drag. encounters→2.
- `#3-login-pie-again` `OPEN` reporter - Seen again, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin clone `61c243f5`, PIE on `L_Core` at the login screen: the first `drive.observe {interactables_only:true, screenshot:false}` returned `AMBIGUOUS_LIVE_ROOT` naming `JoinLeaveHUDNativeOverlay, W_OverallUILayout_C_0`; every later observe/click in two sessions carried `instance_name:"W_OverallUILayout"`. Cheap: one retry.
- `#4-listen-pie-race-lobby` `OPEN` reporter - Seen again, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`, listen-server PIE (host + 1 client) in a PDS race lobby: bare `drive.observe {surface:"game", interactables_only:true}` returned `AMBIGUOUS_LIVE_ROOT` naming `JoinLeaveHUDNativeOverlay, W_OverallUILayout_C_0`; retry with `instance_name` worked. Cheap (one retry). Note `root_index:0` then resolved to `VoiceHUDNativeOverlay`, a root the ambiguity error had not listed.
- `#5-walk-every-root-by-default` `IN-REVIEW` developer - "With no `instance_name` / `root_index`, the game-surface resolver now walks every live UMG root of the selected PIE instance's viewport (Slate order = viewport z-order, bottom-most first), tags each element with `root` (backing widget name), assigns handles across the combined set (same `AssignHandles` core, unbounded parent walk, so a single root keeps its old handles and same-named leaves in two roots get `[k]`), and reports `root_name` as the walked roots comma-separated. `instance_name` / `root_index` still narrow to one root through `SelectRootCandidate`; a substring matching several roots stays `AMBIGUOUS_LIVE_ROOT`, now suffixed with the PIE instance. All game-surface verbs (observe, expect, wait_for, click/hover/scroll/type/key/drag) share the path. Note #4's `root_index:0` -> `VoiceHUDNativeOverlay` was the instance flip tracked in `F-drive-pie-instance-selector`, fixed there. Files: `Handlers/Drive/DriveLiveResolver.{h,cpp}` (ResolveRoots/WalkRoots), `DriveTypes.h` (`FDriveElement::Root`), `DriveJson.cpp` (`root`), `DriveHandlerCommon.cpp` (byte estimate), `docs/wiki-src/drive.md`, `CHANGELOG.md`. Tests: `PinWright.drive.game_surface.TwoUmgRootsObservedTogether` (owned host-neutral PIE, two native UUserWidgets with same-named buttons; fails AMBIGUOUS_LIVE_ROOT if reverted)."
- `#6-test-releases-probes-before-pie-end` `IN-REVIEW` developer - "Full suite (offscreen, Linux): `PinWright.drive.game_surface.TwoUmgRootsObservedTogether` passed its assertions but tripped the end-of-PIE leak ensure (`B_DroneGameInstance_C ... not cleaned up by GC`, referenced through the probe `TestWidgetWithStructBIE`): the test held its two PIE-outered probe widgets in `TStrongObjectPtr`s until the post-PIE cleanup command. Test-only fix: the run command now `RemoveFromParent`s and resets both strong refs before `FEndPlayMapCommand` (and on the creation-failure path). Product code unchanged. File: `Tests/Drive/TestDriveGameSurfaceSelection.cpp`."
- `#7-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.drive.game_surface.TwoUmgRootsObservedTogether` passed in w23-final with no PIE leak ensure (owned host-neutral PIE, two native UUserWidgets with same-named buttons): a bare observe walks both roots, tags each element with `root`, and same-named leaves get distinct handles; `PieInstanceSelection` also passed. The proposed fix is met. Limit: not re-run in a PDS PIE with `JoinLeaveHUDNativeOverlay` and `W_OverallUILayout_C_0`.
