---
id: F-game-features-plugin-management
title: "Game Feature plugins: list registered plugins with lifecycle state (read-only)"
status: IN-REVIEW
severity: Medium
category: feature
tags: [game-features, plugins, lyra, parity-ue58]
---

# Game Feature plugins: list registered plugins with lifecycle state (read-only)

Zero `GameFeature` matches across PinWright `Source\` (verified). The `game_framework.*` namespace (`Handlers/Systems/GameFrameworkHandler.cpp`) manages GameMode/GameState Blueprints and never touches the Game Features subsystem — unrelated. Lyra-derived projects (including the host PDS project) gate whole gameplay slices behind Game Feature plugins; agents cannot see which GF plugins exist or what lifecycle state each is in, so a whole class of "why is this feature not running" debugging is invisible.

UE 5.8 parity evidence: GameFeaturesToolset (`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\GameFeaturesToolset\`), over `UGameFeaturesSubsystem`.

## Scope (this ticket)

`game_features.list()` — read-only enumeration of every **registered** Game Feature plugin, each with `name`, `url`, `state` (Installed/Registered/Loaded/Active plus transition/error states), and `loadedAsBuiltIn`. Backed by the public `UGameFeaturesSubsystem::ForEachGameFeature(TFunctionRef<void(FGameFeatureInfo&&)>)` + `UE::GameFeatures::ToString(EGameFeaturePluginState)`.

Implemented as an optional-engine-module handler (canonical `Handlers/Water/WaterHandler.cpp`): `REGISTER_RPC_HANDLER` is unconditional (namespace appears in the wiki), the body is `__has_include("GameFeaturesSubsystem.h")`-gated, `Build.cs` soft-links the module via `TryAddConditionalModule(..., "GameFeatures", "GameFeatures")`, and `PinWright.uplugin` enables the GameFeatures plugin (it ships with the engine at `Plugins/Runtime/GameFeatures` but is disabled by default) so the module loads and the `UGameFeaturesSubsystem` engine subsystem is actually created at runtime. When the subsystem/module is absent it returns `GAME_FEATURES_NOT_AVAILABLE` — never a fabricated empty list.

## Acceptance (host-viable)

`game_features.list` is registered and, when invoked, returns success with a `plugins` array. On this fuzzing host (`EAContentExamples57`, which enables no Game Feature plugins) that array is **empty** but the field is present; on a GameFeatures-enabled host it is populated with `{name, url, state}` entries. No Lyra/ShooterCore content is referenced (agent-conventions.md forbids Lyra content/modules in tests). Regression test: `PinWright.game_features.list.RegisteredAndListsPlugins` (`Tests/World/TestGameFeaturesListHandler.cpp`) — registration + invoke-success + `plugins`-array-present, all red pre-fix because no `game_features.*` handler is registered.

## Split off (need a GameFeatures-enabled host to verify)

`game_features.set_state(plugin, state)` (drive the async/latent state machine) and `game_features.get_actions(plugin)` (actions declared by the GF data asset + applied status) were moved to **F-game-features-set-state-and-actions**. Both require a live GF plugin fixture to exercise their real (non-error) behavior, which no fuzz host provides, so neither can satisfy the differential red→green regression gate here; `set_state` is additionally an async mutation whose success path would ship unverified on this host.

## History
- `#1-gamefeatures-invisible` `OPEN` reporter — Game Feature plugins are entirely invisible to the RPC surface (zero source matches) despite Lyra-based hosts gating gameplay on them. Epic 5.8 GameFeaturesToolset (7 tools) is the parity target; add list/set_state/get_actions.
- `#2-reword-readonly-list` `IN-REVIEW` developer — REWORD + implemented. Verified the gap is real (red test reproduced) but the original acceptance ("list shows ShooterCore et al … deactivate + reactivate a test feature round-trips") is Lyra-specific (forbidden by agent-conventions.md) and untestable on this GF-disabled host, so narrowed this ticket to the fully-verifiable read-only `game_features.list` (registered GF plugins + lifecycle state) and split the async `set_state` + `get_actions` (need a GF-enabled host / live GF plugin) to F-game-features-set-state-and-actions. Severity Medium unchanged. Shipped `game_features.list` in `Handlers/Systems/GameFeaturesHandler.cpp` (optional-engine-module handler over `UGameFeaturesSubsystem::ForEachGameFeature`), registered `ERR_GAME_FEATURES_NOT_AVAILABLE` in `Handlers/ErrorCodes.h`, soft-linked GameFeatures in `PinWright.Build.cs` and enabled it in `PinWright.uplugin` (so the engine subsystem is created). Adopted the red test `PinWright.game_features.list.RegisteredAndListsPlugins` (`Tests/World/TestGameFeaturesListHandler.cpp`): builds clean, and the test is now GREEN (subsystem initializes; empty `plugins` array on this host, field present) — differential red→green.
- `#3-test-phase-fix` `IN-REVIEW` developer — Test-phase finish (fuzz2): compiled clean and the full PinWright suite (3626 tests) went green with zero failures/crashes; the adopted test `PinWright.game_features.list.RegisteredAndListsPlugins` executed and asserted (registration + invoke-success + `plugins`-array-present; empty-events = passing signature). Diff-review hardening is now part of the shipped `GameFeaturesHandler.cpp`: the body gate was tightened from a bare `__has_include("GameFeaturesSubsystem.h")` to `__has_include("GameFeaturesSubsystem.h") && UE_VERSION_NEWER_THAN_OR_EQUAL(5, 4, 0)` (added `#include "Misc/EngineVersionComparison.h"`), so the shared UE 5.3 build takes the honest `GAME_FEATURES_NOT_AVAILABLE` `#else` path (5.3 ships GameFeatures under `Plugins/Experimental`, which `TryAddConditionalModule` links so `__has_include` alone is true, but 5.3 exposes neither `FGameFeatureInfo` nor `ForEachGameFeature`). On 5.4+ (the 5.7 host, 5.6 CI) compiled behavior is byte-identical. No test added or changed this phase. Note: with GameFeatures enabled in `PinWright.uplugin`, this host actually loads engine built-in GF plugins, so the live `plugins` array is populated rather than empty — the key-agnostic test asserts presence only, so it stays green either way and the handler enumeration is exercised against a live, non-empty subsystem.
