---
id: E-drive-observe-multi-root-default
title: "drive.observe on the game surface fails AMBIGUOUS_LIVE_ROOT whenever two UMG roots are on the viewport (PDS: JoinLeaveHUDNativeOverlay + W_OverallUILayout_C_0), so every observe needs instance_name"
status: OPEN
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
