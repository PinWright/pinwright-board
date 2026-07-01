---
id: B-setup-screen-restores-ignoring-launch-flag
title: "Setup screen reopens every launch despite Show-on-Launch unchecked"
status: IN-REVIEW
severity: Medium
category: bug
tags: [setup-screen, tabs, layout, ux]
---
# Setup screen reopens every launch despite Show-on-Launch unchecked

With **Show Setup Screen on Launch** unchecked and no port conflict, the Editor
Automation setup screen still opens on every editor start. Unchecking the box,
closing the tab, and even cleanly exiting do not stop it.

Root cause: the setup screen is a standalone-window `NomadTab`
(`ETabRole::NomadTab`, id `EditorAutomationSetupScreen`). UE persists
standalone nomad-tab windows in the **machine-global** editor layout
(`%LOCALAPPDATA%\UnrealEngine\Editor\EditorLayout.json`, shared across every
engine version and every project checkout), and `FGlobalTabmanager::RestoreFrom`
recreates that window synchronously at startup — before
`OnMainFrameCreationFinished` fires and entirely independent of
`bShowSetupScreenOnLaunch`, which only gated the plugin's own `TryInvokeTab`.
Because the layout file is global, a clean exit that writes `ClosedTab` in one
editor is clobbered back to `OpenedTab` by any other editor instance/version
that has the tab open, so manual cleanup never sticks. Verified by reading the
live CDO value (`bShowSetupScreenOnLaunch=false`) off the running editor while
the tab was open with a green "listening" banner.

**Fix:** Register the nomad tab spawner with a stateful `FCanSpawnTab` predicate
(`CanSpawnSetupTab`) returning a `bAllowSetupTabSpawn` gate that starts `false`.
The gate blocks UE's startup layout-restore from recreating the window;
`OnMainFrameCreationFinished` flips it `true` after restore so the explicit
launch/conflict `TryInvokeTab` path and later manual Tools-menu opens still work.
The same predicate also gates the menu entry, which is why a constant `false`
would be wrong. The persisted global-layout state becomes irrelevant on every
launch. All changes are in `EditorAutomationRpcGatewayModule.cpp`.

## History
- `#1-initial-repro` `OPEN` reporter — Setup screen auto-opens on every launch with Show-on-Launch unchecked and no port conflict; live CDO confirms the flag is false, so UE's global-layout nomad-tab restore is reopening it independent of the flag.
- `#2-canspawntab-gate` `IN-REVIEW` developer — Added stateful `CanSpawnSetupTab`/`bAllowSetupTabSpawn` gate passed to `RegisterNomadTabSpawner` in `EditorAutomationRpcGatewayModule.cpp`; gate is false during startup layout-restore (blocks auto-restore) and flipped true in `OnMainFrameCreationFinished` (preserves launch/conflict auto-open and manual menu open). Needs a rebuild + manual verification in a live editor (no unit coverage possible for the tab-restore path).
