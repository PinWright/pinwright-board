---
id: B-transient-test-blueprints-compiled-every-pie
title: "Transient test and scratch Blueprints are RF_Standalone and never discarded, so every PIE start recompiles hundreds of them; an orphaned interface implementer trips ensure(Owner->SkeletonGeneratedClass)"
status: IN-REVIEW
severity: High
category: bug
tags: [tests, pie, fixture-leak, blueprint, bpir, find-node-types, ensure, gap-analysis-2026-09-28]
encounters: 2
costly: 2
lastSeen: 2026-09-29T12:00:00Z
---

# Leaked transient Blueprints are recompiled at every PIE start

The drive.observe owned-PIE start hit a handled ensure `Owner->SkeletonGeneratedClass`
(`BlueprintCompilationManager.cpp:3954/3553`; crash reports
`Saved/Crashes/UECC-Windows-FD193B3A403E5258EEF99EB91A925A00_0000/_0001`) inside
`FInternalPlayLevelUtils::ResolveDirtyBlueprints`. That function compiled 138 leaked
`/Engine/Transient` Blueprints, and failed on `/Engine/Transient.BpirInterfaceExplicitOverride_0`.

Root causes:
1. **Test helpers leak by construction.** `CompilerTestUtils::CreateTransientTestBP` / `...WithParent`
   (`Tests/Bpir/CompilerTestUtils.h`, ~450 call sites in 69 files) call
   `FKismetEditorUtilities::CreateBlueprint`, which flags the Blueprint `RF_Standalone`. The helper
   comment "cleaned up by GC" is false: nothing discards them, so they live for the whole session
   and stay dirty. `ResolveDirtyBlueprints` recompiles every valid, not-up-to-date, non-data-only
   Blueprint at each PIE start.
2. **Orphaned interface implementer.** `PinWright.bpir.compiler.integration.InterfaceNamedOutputsOverride`
   (`Tests/Bpir/TestBpirInterfaceOutputOverride.cpp:367-370`) makes two transient implementers of a
   saved interface Blueprint. Its only teardown was `CleanupTestAsset(InterfacePath)`, which removes
   the interface's generated classes, while the implementers stayed alive. PIE later compiled an
   implementer against the dead interface and ensured.
3. **Production leak with the same mechanism.** `blueprint.graph.find_node_types` and
   `blueprint.graph.get_node_type_pins` (`Handlers/Blueprint/BlueprintNodeDiscoveryHandler.cpp`,
   `MakeScratchProbeGraph`) create a `PinWrightNodeProbe_N` scratch Blueprint per call the same
   way. The code comment claims it is "collected as transient garbage"; it is never collected.
   In a user's session each call adds one more Blueprint to every PIE start.

`editor.play`'s own test (`Tests/EditorOps/TestEditorHandlers.cpp:711`) already skips under
`-unattended` citing this leak.

**Fix:** see History.

## History
- `#1-pie-compiles-leaked-bps` `OPEN` reporter — Filed from a verification run of the drive.observe fix: handled ensure at owned-PIE start, 138 leaked transient Blueprints compiled. Root-caused to the three items above. Cost: one verification run carried a false ensure/failure.
- `#2-discard-at-test-end-and-after-use` `IN-REVIEW` developer — (a) `Tests/AutomationSuiteMaintenance.cpp` `OnTestEnd` (PinWright tests only) now calls `DiscardLeakedTransientBlueprints`. It takes every valid UBlueprint whose package is under `/Engine/Transient` and whose generated class has no live instance, closes any asset editor on it, then applies `PwTestAssetTeardown::DiscardLoadedAssetNoGc` (clears `RF_Standalone`, removes the generated classes, detaches) and `MarkAsGarbage`. PIE skips it immediately (`!IsValid`) and the next suite reset reclaims it. No force-delete, no GC dependency. One `.cpp` covers all ~450 helper call sites; `CompilerTestUtils.h` is untouched. (b) `TestBpirInterfaceOutputOverride.cpp`: a scope-exit declared after the interface cleanup discards both implementers first. (c) `BlueprintNodeDiscoveryHandler.cpp`: both handlers discard their scratch Blueprint on scope exit (`ClearFlags(RF_Standalone|RF_Public)` + `MarkAsGarbage`); comment corrected. `-SingleFile` compile of all three: clean. Not changed: the `editor.play` unattended skip, which also guards host-GameMode PIE for the production verb.
- `#3-fixtures-under-scratch-root` `IN-REVIEW` developer — Side finding fixed: test fixtures that saved outside the scratch root now live under it, so zz_suite_end sweeps them. Every `/Game/<root>` literal in test sources, for roots that left directories in host `Content/`, now reads `/Game/PinWrightTests/<root>`. Roots: `__PW_GatewayTests` (including this file's two tests), `__PW_AssetSaveDiskGuard`, `__PW_DeleteTests`, `__PW_DumpTests`, `__PW_MoveTests`, `__PW_RenameDuplicateTests`, `__PW_SCRevertTests`, `PwModelHandlerTests`, `__McpTest__`, `_Test`, `TestFolder`, `UnitTest`. That is 114 files, all under `Source/**/Private/Tests/`, six of them test-only headers (AssetRefDirectionFixtures.h, MaterialShaderStateTestFixtures.h, MaterialTestHelpers.h, AssetDumpMismatchedNameFixture.h, WidgetTestFixtures.h, WidgetXmlTestHelpers.h). Literal-only change; no production code or docs referenced these roots. Left as is: `/PinWright/__PW_SoftClassTests` (plugin mount, never saved), plus never-written "missing/absent" example paths and the production default `/Game/GeneratedMeshes`. `-SingleFile` compile of all 108 .cpp in six batches: clean. Removed empty, untracked host dirs: the seven `__PW_*` except `__PW_GatewayTests`, `PwModelHandlerTests`, `TestFolder`, `UnitTest`, `_Test`, `GeneratedMeshes` and 16 recreated `MCP_*Probe`. `__PW_GatewayTests` and `__McpTest__` (also empty and untracked) were left because the running full suite had written them minutes earlier.
- `#4-sweep-gutted-template-cache` `IN-REVIEW` developer — Returned by the full offscreen suite (`Saved/PinWright/test-runs/ed6be6c8dd3543b49ee4de940a39a39b/automation.log`, died at 1743/5388, crash `UECC-Windows-624902454679BAEE01D81D97CDF4827B_0001`). It hit `check(InContext.Struct)` (`PropertyAccessEditor.cpp:203`) via `UK2Node_PropertyAccess::AllocatePins` <- `UBlueprintNodeSpawner::Prime` <- `FBlueprintActionDatabase::Tick`, one tick after the #2 sweep ran; `_0000` was the same mechanism as `ensure(SelfScope)` via `UK2Node_AddComponentByClass`. Root cause, confirmed in engine source: the sweep's predicate also matched the editor's own `FBlueprintNodeTemplateCache` Blueprints. Those are `PROTO_BP_*`, created in the transient package with `RF_Transient` (`BlueprintNodeTemplateCache.cpp:179-185`) and have no class instances. Every spawner's template node lives in them, so removing their generated classes left the next `Prime` without a skeleton class. The action-database-entry theory does not hold for these fixtures: transient-package objects are never database keys (`IsObjectValidForDatabase` requires `IsAsset()`, which excludes the transient package, `Obj.cpp:2788`). Fix in `Tests/AutomationSuiteMaintenance.cpp`: (a) a candidate must be created by the finished test (snapshot on `OnTestStartEvent`), not `RF_Transient`, and not `PROTO_BP_*`; (b) the discard now mirrors the engine's `FBlueprintUnloader::UnloadBlueprint`. It flushes the compilation queue, closes editors, calls `ClearEditorReferences()` (`OnBlueprintUnloaded` / `OnBlueprintGeneratedClassUnloaded`: action database, Blueprint editors, Find-in-Blueprints, thumbnails, component type registry, editor utility subsystem), then `ClearAssetActions` on the Blueprint and both class keys, then `DiscardLoadedAssetNoGc` + `MarkAsGarbage`, and resets the transaction buffer when it referenced a discarded Blueprint. `BlueprintNodeDiscoveryHandler.cpp` scratch discard now uses the engine unloader's exact sequence (flags, `MarkAsGarbage`, `ClearEditorReferences`) and touches only its own scratch Blueprint. `TestBpirInterfaceOutputOverride.cpp` also calls `ClearEditorReferences` first. `-SingleFile` compile of all three: clean.
- `#5-scratch-root-guard-idea` `IN-REVIEW` reporter — No status change. The scratch-root relocation in `#3` (and the `MCP_*Probe` half in `B-statetree-dump-fixture-breaks-later-pie` `#3`) fixed the known literals, but nothing stops the next test from saving outside `/Game/PinWrightTests` again: `Tests/AutomationSuiteMaintenance.cpp` has no save hook and no static check exists. Guard idea, for whoever verifies or reopens: (a) runtime, in `AutomationSuiteMaintenance`, subscribe to `UPackage::PackageSavedWithContextEvent` while a `PinWright.*` test runs and fail that test (`AddError`, naming the package) when a saved package is neither under `PinWrightSuiteMaintenance::ScratchRootPackagePath()` nor transient; (b) static, a `check_test_skips.py`-style scan of `Source/**/Private/Tests/` flagging `/Game/<root>` literals whose root is not `PinWrightTests`, with a site-comment opt-out for never-written example paths. (a) catches paths built at runtime; (b) catches them before a build. Verification of `#3` should also confirm the host's `Content/` gains no new top-level directory after a full suite.
