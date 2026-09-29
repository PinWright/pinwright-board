---
id: B-resolve-class-by-name-logs-loadasset-error
title: "ResolveClassByName calls UEditorAssetLibrary::LoadAsset on any path-shaped input, logging an engine Error for every missing path and on every call in PIE"
status: IN-REVIEW
severity: Medium
category: bug
tags: [class-resolution, log-noise, pie, actor, gap-analysis-2026-09-28]
encounters: 2
lastSeen: 2026-09-29T16:40:00Z
---

# ResolveClassByName logs engine errors for misses

`ResolveClassByName` (`Utils/ClassUtils.cpp`) is shared by every verb that takes a class or
Blueprint path (`actor.spawn_from_blueprint`, `actor.find_by_class`, capture label filters and
more). For any input containing `/` (other than `/Script/`) it called `UEditorAssetLibrary::LoadAsset`
unconditionally, even though the fallbacks below it can resolve the name. `LoadAsset` logs
`LogEditorAssetSubsystem: Error: LoadAsset failed: The AssetData '<path>' could not be found in the
Asset Registry.` for every miss, and `LogUtils: Error: The Editor is currently in a play mode.` on
every call during PIE, where it returns nothing. So a user who passes a wrong path gets an engine
Error in their log beside the verb's own typed `CLASS_NOT_FOUND`. It also fails any automation
test that exercises the miss. `actor.spawn_from_blueprint.WithPath` had hidden this behind a blanket
`bSuppressLogErrors` (full offscreen log `Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log`).

**Fix:** see History.

## History
- `#1-found-in-suppression-audit` `OPEN` reporter — Found by the blanket log-suppression audit (`B-tests-blanket-log-suppression`).
- `#2-quiet-package-probe` `IN-REVIEW` developer — `ResolveClassByName` now calls `LoadAsset` only when the package is present. The check is quiet: `FPackageName::IsValidLongPackageName`, then `FindPackage` or `FPackageName::DoesPackageExist`, and never while `PinWrightPieState::IsPlayInEditorActive()`. Results are unchanged: `LoadAsset` returned nothing in both skipped cases, and the lookups below still run. `actor.spawn_from_blueprint.WithPath` no longer needs any suppression.
- `#3-package-probe-was-fooled` `OPEN` tester — The full offscreen suite (`Saved/PinWright/test-runs/7b2d4d4c5a494988a2c90b28389aad29/automation.log`, `actor.spawn_from_blueprint.WithPath`) still logged `LoadAsset failed ... '/Game/Blueprints/BP_TestActor.BP_TestActor'`. Cause: the spawn handler's `LoadBlueprintAsset` runs first. Its `ResolveAsset` step calls `LoadObject` on the missing path, which warns "Failed to find object" and leaves an empty in-memory `UPackage` behind. The `FindPackage` half of the #2 probe then read that package as present and let the call through to `LoadAsset`.
- `#4-probe-the-registry-like-loadasset-does` `IN-REVIEW` developer — `ResolveClassByName` now asks the Asset Registry the same question `LoadAsset` does: `IAssetRegistry::GetAssetByObjectPath` on the normalized object path, with `IsValidLongPackageName` first. `LoadAsset` runs only when that succeeds and PIE is not active; any other path would have made `LoadAsset` log its Error and return nothing. The test is rewritten to what it was named for: a missing Blueprint path under the scratch root must be refused with the typed `CLASS_NOT_FOUND`. It uses a capture, so the "response dropped" warning is gone, and the framework's error check pins "no engine error". It no longer references the host path `/Game/Blueprints/BP_TestActor`. The "Failed to find object" warning from `LoadBlueprintAsset`'s `LoadObject` remains; it is a warning. `-SingleFile` compile of `ClassUtils.cpp` and `TestActorHandlers.cpp`: clean.
- `#5-no-load-attempt-on-a-miss` `IN-REVIEW` developer — One step earlier on the same path: `LoadBlueprintAsset` -> `ResolveAsset(path, /*bLoadObject=*/true)` ran `LoadObject<UObject>` on a path neither the registry nor memory held. That logged the user-visible `Failed to find object 'Object <path>.<name>'` on every wrong Blueprint path and left the empty `UPackage` behind that fooled the #2 probe. `ResolveAsset` (`Utils/AssetUtils.cpp`, shared by 38 handler files) now attempts the load only when the registry matched or `FPackageName::DoesPackageExist` finds the package on disk (a file not yet indexed). In-memory objects, transient ones included, are still found by the `FindObject` step that runs first. Results are identical, because every skipped load was a guaranteed miss. The one existing exact expectation that counted this line (`TestNestedParamKeyGate` `NestedInputClosure.ConfirmedCasesRefuseBeforeLoad`, `'Object /Game/DoesNotExist/BP_Missing.BP_Missing'`) was removed. The typed `LoadObject<UBlueprint>` / `<UMaterial>` / `<UMaterialInstanceConstant>` warnings it also declares come from handler code, not `ResolveAsset`, and stay. `actor.spawn_from_blueprint.WithPath` now also asserts zero `Failed to find object` lines while the handler runs. It uses a scoped `FOutputDevice` on `GLog` that counts the needle and declares nothing, so the check holds on hosts that suppress warnings and the automation framework's own capture is left untouched. `-SingleFile` compile of `AssetUtils.cpp`, `TestActorHandlers.cpp`, `TestNestedParamKeyGate.cpp`: clean.
