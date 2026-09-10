---
id: E-suite-runs-on-host-startup-map
title: "The automation suite runs against the host project's startup map, so any unguarded world edit reaches host content"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [testing, automation-suite, host-content, editor-world, new-map, hygiene]
---

# The automation suite runs against the host project's startup map

UE automation runs against whatever map the host project opens at editor startup
(`EditorStartupMap`). For this plugin that means the world every test spawns into,
edits, dirties and occasionally saves **is a shipped host asset** — on the PDS host,
`/Game/System/FrontEnd/Maps/L_Core`. A full suite run therefore ends with the host
project's `git status` dirty, and a `Content/` diff nobody authored.

The existing containment is per call site and therefore opt-in:

- `PinWrightSuiteMaintenance::FScopedForeignDirtyPackageSuspension` scopes the dirty
  set of a save-everything verb (`editor.save_all`) to the fixture scratch root, but
  only where a test remembers to wrap the call.
- `FScopedEditorWorldActorGuard` destroys actors a test spawned and restores the
  persistent level's dirty flag, but only where a test remembers to construct it.

Neither covers the general case: a test that dirties the editor world through any
other path — a component edit, a level property, a verb that touches the world
without spawning an actor — leaves the host's map modified, and any later flush
writes it.

## What it should do

Remove the target rather than guarding each path to it. Open a **blank, untitled,
unsaveable** world before the rest of the suite runs, so the world every test edits
has no package in the host's content tree at all. `GEditor->NewMap(false)` produces
exactly that: its package is `/Temp/Untitled_<N>`, and `/Temp/` is a read-only mount
rooted at `<Project>/Saved`, with no file on disk for a save to overwrite.

**Fix:** a first-sorted suite-start step, the mirror of the existing suite-end
scratch-root gate (`PinWright.zz_suite_end.ScratchRootIsEmptyOnDisk`).

## History
- `#1-suite-runs-on-host-map` `OPEN` reporter — The suite runs on the host's configured startup map, so unguarded editor-world edits reach host content: the world under test is a shipped `/Game/` asset, the two existing containments (`FScopedForeignDirtyPackageSuspension`, `FScopedEditorWorldActorGuard`) are opt-in per call site, and anything that dirties the world outside them leaves the host project's `Content/` modified. Sibling of `B-suite-run-dirties-host-repo`, which closed the `editor.save_all` and scratch-root leaks; this is the remaining structural one — the world itself. Asks for a suite-start step that swaps the editor world for a blank transient world so there is no host asset for a test to dirty in the first place.
- `#2-suite-start-blank-transient-world` `IN-REVIEW` developer — Added `PinWright.aa_suite_start.OpenBlankTransientWorld` in `Source/PinWright/Private/Tests/Infra/TestSuiteStartBlankWorld.cpp`: it records the startup world's package name for the report, runs the shared pre-swap survivor probe (`PinWrightMapSwapGuard::ProbeResidentWorldSurvivors(FString(), /*bTransactionBufferWillBeCleared=*/false)` — the same guard `level.create` reaches through `MapSwapGuardRefusal.h`), calls `GEditor->NewMap(false)`, and asserts the resulting editor world's package is untitled-or-transient (`FPackageName::IsTempPackage` / `RF_Transient`), carries `PKG_NewlyCreated`, and is no longer the startup map. It saves nothing and cannot prompt: `NewMap` (`EditorServer.cpp:2187`) is the bare form, while the prompting `CreateNewMapForEditing` (`:2145`) is the one that calls `FEditorFileUtils::SaveDirtyPackages` at `:2158`. `PINWRIGHT_ASSERTIONS_SKIPPED` is emitted via `PinWrightTestSkip::SkipAssertions` when there is no editor world, and when the survivor probe refuses or cannot measure — a refusal leaves the run on the host map, which must be visible rather than silent. The `aa_` second segment is the ordering mechanism mirroring the `zz_` gate: the controller sorts the batch by display name with a case-insensitive compare (`AutomationControllerManager.cpp:1042-1051`) and the earliest other second segment in the tree is `actor`, verified with `Content/Python/check_test_ids.py` (CLEAN, 5333 ids) and a full-tree second-segment scan (nothing sorts before `aa_suite_start`). A suite audit for tests that assume a saved editor world found four consequences, all fixed here. (a) `FScopedEditorWorldMapGuard` (`Source/PinWright/Private/Tests/TestWorldUtils.h`) restored the original map only when its package exists on disk, which an untitled world never does — a skipped restore would have left the editor world context on the throwaway probe map those tests destroy in their own teardown (`level.structure.create_level.MakesWorldActive` and the shared `FScopedProbeWorld`, plus six `level.create` / `level.save` / `lighting.create_lighting_enabled_level` probes), so the guard now falls back to opening an equivalent blank world through the same `NewMap` + probe pair. (b) `PinWright.editor.open_asset.WorldDoesNotCrashEditor` (`Tests/EditorOps/TestOpenAssetWorldNoCrash.cpp`) would have gone hard red: its fallback fixture hardcoded `/Game/Maps/ExampleProjectWelcome`, a map from a different host project that does not exist here, and the untitled world no longer satisfies its `/Game/` + `DoesPackageExist` gate. It now discovers a map from the asset registry, and its `FScopedEditorWorldMapGuard` moved above the resolve so the host map it opens is not left ambient for the rest of the run. (c) `PinWright.level.load.DefersMapSwapToSafePoint` (`Tests/World/TestLevelLoadSafePoint.cpp`) read its swap target off the ambient world and would have taken its `fixture-missing` skip, silently dropping the whole safe-point counterfactual and emitting `PINWRIGHT_ASSERTIONS_SKIPPED` — which alone downgrades a run to `COMPLETED_WITH_SKIPS` / exit 1; it now prefers the ambient map when saved and falls back to a registry-discovered one, skipping only on a host with no map asset at all. (d) `FindAlternateOnDiskMap` moved from file scope in `Tests/World/TestLevelHandlers.cpp` to `Tests/TestWorldUtils.h` so those three call sites share one definition instead of three registry scans. Docs: `Docs/test-organization.md` gained a **The suite-start blank world** subsection beside the scratch-root rules (including the scoped-run limitation and the discover-don't-hardcode rule for map fixtures) and `CLAUDE.md` a one-paragraph Testing note; the stale "EXACTLY ONE test calls ProbeResidentWorldSurvivors" claim in `Tests/Core/TestMapSwapWorldSurvivorProbe.cpp` was corrected to name the two non-asserting callers. Counterfactual: revert the `GEditor->NewMap(false)` call and the editor world's package is still the host startup map, so the untitled-or-transient assertion fails on that package name. Not compiled or run (orchestrator owns the build).
