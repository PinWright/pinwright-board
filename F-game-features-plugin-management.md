---
id: F-game-features-plugin-management
title: "Game Feature plugins: list, enable/disable, action state readback"
status: OPEN
severity: Medium
category: feature
tags: [game-features, plugins, lyra, parity-ue58]
---

# Game Feature plugins: list, enable/disable, action state readback

Zero `GameFeature` matches across PinWright `Source\` (verified). The `game_framework.*` namespace manages GameMode/GameState classes, unrelated to the Game Features subsystem. Lyra-derived projects (including the host PDS project) gate whole gameplay slices behind Game Feature plugins; agents cannot list them, toggle their state, or read which actions are active, so a whole class of "why is this feature not running" debugging is invisible.

UE 5.8 parity evidence: GameFeaturesToolset (7 tools in the 5.8.0 release build, `C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\GameFeaturesToolset\`), over `UGameFeaturesSubsystem`.

Proposed scope:
- `game_features.list()` - registered GF plugins with URL, current state (Installed/Registered/Loaded/Active), and plugin metadata.
- `game_features.set_state(plugin, state)` - drive the state machine (activate/deactivate/load/unload), async via job where transitions are latent.
- `game_features.get_actions(plugin)` - actions declared by the GF data asset and their applied status.

Acceptance: on the host project, list shows ShooterCore et al with correct states; deactivate + reactivate a test feature round-trips state and its actions reapply.

## History
- `#1-gamefeatures-invisible` `OPEN` reporter — Game Feature plugins are entirely invisible to the RPC surface (zero source matches) despite Lyra-based hosts gating gameplay on them. Epic 5.8 GameFeaturesToolset (7 tools) is the parity target; add list/set_state/get_actions.
