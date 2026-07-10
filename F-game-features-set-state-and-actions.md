---
id: F-game-features-set-state-and-actions
title: "Game Feature plugins: set_state (drive lifecycle) + get_actions (action readback)"
status: OPEN
severity: Medium
category: feature
tags: [game-features, plugins, lyra, parity-ue58]
---

# Game Feature plugins: set_state (drive lifecycle) + get_actions (action readback)

Split from **F-game-features-plugin-management**, which shipped the read-only `game_features.list` verb (enumerate registered GF plugins + lifecycle state). Two verbs remain unimplemented:

- `game_features.set_state(plugin, state)` — drive the GF state machine (activate/deactivate/load/unload) via `UGameFeaturesSubsystem::ChangeGameFeatureTargetState(URL, EGameFeatureTargetState, CompleteDelegate)`. Transitions are **latent/async** (delegate completion), so the success path must run through a job (`Ctx.StartJob(...)` / async token; clients poll `system.job_status`). Synchronous validation (unknown plugin name → `GAME_FEATURE_PLUGIN_NOT_FOUND`, bad state string → `INVALID_PARAMS`) is testable anywhere; the async round-trip is not.
- `game_features.get_actions(plugin)` — actions declared by the plugin's `UGameFeatureData` and their applied status. Resolve name→URL (`GetPluginURLByName`), then `GetGameFeatureDataForActivePluginByURL(URL)` → `UGameFeatureData::GetActions()` (`const TArray<UGameFeatureAction*>&`). Meaningful output requires an **active** GF plugin.

## Why split (testability gate)

Both verbs' real (non-error) behavior needs a GameFeatures-enabled host with at least one live GF plugin. The fuzzing hosts (`EAContentExamples57`) enable **no** Game Feature plugins, so on them `set_state`/`get_actions` can only ever hit their not-found/invalid error paths — the differential red→green regression gate cannot prove a state round-trip or a populated actions list here. `set_state` additionally is an async mutation whose success path must not ship unverified. agent-conventions.md forbids Lyra/ShooterCore fixtures and forbids `return true` skips, and a real GF fixture is a `.uplugin` dir + `UGameFeatureData` asset (not buildable in-code cheaply).

## Prerequisite

Pick this up on a host that enables the GameFeatures plugin AND ships (or authors in-code) a minimal GF plugin fixture — or land a shared in-code GF fixture helper first. Without that, only the synchronous input-validation error paths are testable, which does not prove the round-trip and is insufficient for GO.

## Implementation notes (same conventions as the shipped `list` verb)

Optional-engine-module handler in the same file/namespace as `game_features.list` (`Handlers/Systems/GameFeaturesHandler.cpp`): unconditional `REGISTER_RPC_HANDLER`, `MCP_HAS_GAMEFEATURES` body gate, module already soft-linked in `Build.cs`. Wrap the mutation in `FScopedTransaction`. Register any new `SendError` codes in `Handlers/ErrorCodes.h` (the `PinWright.core.error_codes.AllEmittedCodesAreRegistered` suite test scans emit sites).

## History
- `#1-split-from-list` `OPEN` developer — Split off from F-game-features-plugin-management (which shipped read-only `game_features.list`). set_state (async lifecycle mutation) + get_actions (action readback) both require a GameFeatures-enabled host with a live GF plugin to satisfy the differential regression gate; deferred here rather than shipping an unverifiable async path on the GF-disabled fuzz host.
