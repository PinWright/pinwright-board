---
id: F-game-features-live-fixture
title: "In-code Game Feature plugin fixture: register + drive a synthetic GF plugin through its lifecycle for tests"
status: IN-REVIEW
severity: Medium
category: feature
tags: [game-features, plugins, test-fixture, testing]
claimedBy: fuzz2
claimedAt: 2026-07-13T09:33:37.7160174+03:00
---

# In-code Game Feature plugin fixture (register + drive a synthetic GF plugin for tests)

PinWright's `game_features.*` surface can only be regression-tested against a **live, registered Game Feature plugin**, which no fuzz host provides. Verified against UE 5.7 engine source: a plugin is enumerated by `UGameFeaturesSubsystem::ForEachGameFeature` (and resolvable via `GetGameFeatureDataForActivePluginByURL`) only if it passes `UGameFeaturesSubsystemSettings::IsValidGameFeaturePlugin` — its descriptor must live under a configured game-feature plugins folder — AND carries a `UGameFeatureData` asset. `GameFeaturesSubsystem.cpp:2369` short-circuits everything else as "Not a GFP, trivial success" (no state machine created, nothing enumerated). The `EAContentExamples57` host has zero such plugins: the engine ships zero `UGameFeatureData` assets; the only `ExplicitlyLoaded` engine plugin (GameplayGraph) is `CanContainContent:false` and not a GF plugin; the project enables only PinWright. So `game_features.list` returns an empty array and there is no active plugin whose actions can be read or whose state can be driven.

This ticket asks for a **shared test-only fixture helper** (e.g. `Tests/Systems/GameFeaturesTestFixture.h`) that, within an automation test, brings a synthetic game feature plugin far enough into the `UGameFeaturesSubsystem` that it: appears in `ForEachGameFeature`, resolves via `GetPluginURLByName`, can be driven `Registered -> Loaded -> Active` via `ChangeGameFeatureTargetState` (latent completion), and reads back a non-empty `UGameFeatureData::GetActions()` via `GetGameFeatureDataForActivePluginByURL`. It MUST tear down cleanly (unregister/unmount) so it leaks no state into sibling tests, and MUST NOT reference Lyra/ShooterCore content (agent-conventions.md forbids it).

## Why (what it unblocks)

`F-game-features-set-state-and-actions` (drive lifecycle + action readback) is deferred (`blockedBy` this ticket): both verbs' real, non-error behavior needs a live GF plugin to satisfy the differential red -> green regression gate, and `set_state` is an async mutation whose success path must not ship unverified (agent-conventions.md:47/49). This fixture is the missing infrastructure that makes those verbs testable on a fuzz host.

## Feasibility note

Registering a fully in-memory/transient GF plugin may be infeasible — the subsystem may require an on-disk mounted `.uplugin` under a game-feature folder (`IsValidGameFeaturePlugin` inspects the descriptor path). If so, the fallback is a tiny on-disk GF plugin fixture placed under a test-only path and registered at test start; document whichever path actually works and its teardown.

## Acceptance

A test-only helper exists under `Tests/` that, in a headless run (`-RenderOffScreen -unattended -nopause -nocefaccelpaint`), registers a synthetic GF plugin, activates it through the real `UGameFeaturesSubsystem` API, asserts it reaches `Active` and exposes its declared actions, then unregisters it — with no Lyra content and no leaked state (a subsequent `game_features.list` in the same session returns to the pre-fixture set). No production handler is stubbed or special-cased to satisfy it.

## History
- `#1-need-gf-fixture` `OPEN` developer — Filed as the testability gate blocking F-game-features-set-state-and-actions. Verified against UE 5.7 source that ForEachGameFeature / GetGameFeatureDataForActivePluginByURL enumerate only plugins passing IsValidGameFeaturePlugin (GameFeaturesSubsystem.cpp:2369) that carry a UGameFeatureData; this host has none, so the GF verbs can only ever hit their error paths. Need a shared in-code (or minimal on-disk) GF plugin fixture to drive a real Registered->Active round-trip + populated action readback in an automation test.
- `#2-go-implement-fixture` `IN-REVIEW` developer — GO. Re-verified against UE 5.7 source: only game_features.list ships (GameFeaturesHandler.cpp), and the state machine only exposes a NON-EMPTY GetActions() for a bCanContainContent:true plugin carrying a real on-disk UGameFeatureData (bCanContainContent:false yields a transient EMPTY GFD; GameFeaturePluginStateMachine.cpp:3296-3302), while a content plugin missing that asset errors Registering (:3456). Implementing a test-only fixture that GENERATES a synthetic on-disk GF plugin + a UGameFeatureData carrying one action at test time, drives it Registered->Active via the real UGameFeaturesSubsystem::ChangeGameFeatureTargetState, asserts it reaches Active + GetActions() is non-empty, then deactivates/terminates and deletes the on-disk plugin — no committed asset, no Lyra, no stubbed handler, no leaked state. Co-locating under Tests/World/ (no Tests/Systems/ bucket exists in the taxonomy).
