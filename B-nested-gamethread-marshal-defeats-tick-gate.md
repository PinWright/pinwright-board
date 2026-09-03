---
id: B-nested-gamethread-marshal-defeats-tick-gate
title: "Eleven handler bodies still run inside a redundant AsyncTask(ENamedThreads::GameThread) marshal, which drops them out of the dispatcher's reentrancy guard and makes the SafePoint table unable to gate them — the exact shape that killed the editor on asset.reload"
status: OPEN
severity: High
category: bug
tags: [dispatch, safepoint, tick-safety, rpc-dispatcher, landscape, asset, reentrancy, render-flush, editor-crash-risk, ratchet]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# The marshal that `asset.reload` was fixed for is still on eleven other verbs

`B-asset-reload-access-violation-kills-editor` established that a handler body wrapped in
`AsyncTask(ENamedThreads::GameThread, ...)` is **redundant and actively harmful**, and its fix
removed exactly one instance. The idiom is still present at **fifteen** call sites under
`Source/PinWright/Private/Handlers/` (seventeen lines match the grep; two are the
do-not-re-add comments that fix left behind, at `AssetManageHandler.cpp:1801` and
`LevelHandler.cpp:251`). Eleven of the fifteen are the same defect shape as `asset.reload`.

## Why the marshal is redundant, confirmed from source

`FRpcDispatcher::ProcessRequest` hops to the game thread as its first act
(`Dispatch/RpcDispatcher.cpp:586`) and only then runs the safe-point gate
(`:696-697`), the reentrancy guard (`:745-761`) and the handler. A handler body is therefore
game-thread by construction; a nested `AsyncTask` buys **no thread affinity at all**. What it
changes is position, in three ways, each independently bad:

1. **It leaves the reentrancy guard.** `bProcessingRequest` is set at `RpcDispatcher.cpp:761`
   and cleared in the `ON_SCOPE_EXIT` at `:784` — i.e. when the *handler returns*, which for a
   marshalled body is before any work has happened. Another agent's RPC can then be drained
   *into* the middle of that work. `PendingQueue` exists precisely to prevent this.
2. **It defeats the SafePoint table.** `IsTickUnsafeMethod` is consulted at `:696` against the
   *arrival* stack. A marshalled body lands later on the game thread's named-thread queue —
   the very queue `PinWrightSafePoint::IsInsideNamedThreadPump()` (`Dispatch/SafePoint.h`)
   exists to refuse. So for these verbs a table entry would be decoration. `SafePoint.cpp`
   says this in its own words twice: at `:397-410` (family I, on `landscape.set_material`) and
   at `:456-467` (family J, on why `asset.reload` needed both halves).
3. **It runs outside `FScopedUnattendedRpc`**, whose scope spans the synchronous handler body
   only. One site already works around this by hand (`AssetWorkflowHandler.cpp:63`), which is
   evidence the deferral is understood at that site and unaddressed at the other ten.

## The specific defect this ticket is named for

`SafePoint.cpp:397-410` records the decision **not** to add `landscape.set_material` to
`GTickUnsafeMethodNames`, for two stated reasons — the nested marshal, and the
`environment.build` cross-dispatch that bypasses `ProcessRequest` — and closes with: *"It needs
its continuation routed through `PinWrightSafePoint::DeferToSafePoint` instead - a different
change, in a different file, **filed separately**."* `B-compile-material-not-tick-gated`'s
`## Fix shape` asks for the same decision. **That follow-up was never filed.** No ticket on this
board mentions `DeferToSafePoint`, `RunAtSafePoint` or the marshal in connection with any of
these verbs (checked across all 1701 files). This ticket is the missing filing, widened to the
whole idiom because the same two-reason argument applies verb-for-verb to the other landscape
write verbs and to two `asset.*` verbs.

`landscape.set_material` is not a theoretical case. `LandscapeHandler.cpp:1600` calls
`SetLandscapeMaterialAndNotify`, which runs `ALandscape::PostEditChangeProperty` naming the
`LandscapeMaterial` property (`LandscapeHandler.cpp:312`) — reaching
`UpdateAllComponentMaterialInstances`, one `FComponentRecreateRenderStateContext` per landscape
component and a `FlushRenderingCommands` per new combination MIC. That is hazard family I
verbatim, the family whose shipped evidence is **two editor kills 90 seconds apart on
2026-08-16**, render-thread `EXCEPTION_ACCESS_VIOLATION` under `BeginReleaseResource` with
"FlushRenderingCommands called recursively!" 15 and 21 times in ~170 ms
(`B-compile-material-not-tick-gated` `#2`). The sibling verb that produced those kills is now
gated; this one is not, and cannot be by a table entry alone.

## Every site, classified

Hazard = does the deferred body pump the engine (GC, render flush, slow task, package save,
component (re)registration) where an RPC drained underneath it can mutate the graph, or is it a
short read?

### Group 1 — plain redundant marshal, same shape as the fixed `asset.reload` (11 sites)

| # | file:line | verb | what the deferred body does | hazard | in `GTickUnsafeMethodNames`? |
|---|---|---|---|---|---|
| 1 | `Asset/AssetManageHandler.cpp:1750` | `asset.validate` | `DoesAssetExist` + `LoadAsset` on an already-resident object, then JSON | **none** — explicitly adjudicated harmless at `SafePoint.cpp:469-473` | no, and correctly so |
| 2 | `Asset/AssetMetadataHandler.cpp:237` | `asset.set_tags` | `LoadAsset` + `SetMetadataTag` xN + `MarkPackageDirty` | **low** — a cold `LoadAsset` is a synchronous package load + `PostLoad`; the mutation lands outside the guard | no |
| 3 | `Asset/AssetWorkflowHandler.cpp:58` | `asset.fixup_redirectors` | project-wide asset-registry scan, source-control checkout, `RedirectorFixupPolicy::FixupReferencers(..., bDeleteFixedUpRedirectors=true)` — loads and **re-saves every referencing package**, then deletes assets | **high** | no |
| 4 | `Asset/AssetWorkflowHandler.cpp:1155` | `asset.generate_lods` | per path: `SetNumSourceModels` + `UStaticMesh::Build()` + `PostEditChange()` + `McpSafeAssetSave` | **high** — this is family H.4 verbatim (`SafePoint.cpp:305-309`: the build releases and re-creates render resources, i.e. render fences and `FlushRenderingCommands` on the handler's own stack) | no |
| 5 | `Environment/LandscapeGrassFlushHandler.cpp:68` | `landscape.flush_grass` | `ALandscapeProxy::FlushGrassComponents` per proxy + `ULandscapeSubsystem::RegenerateGrass(bInForceSync=true)` (`GrassTypeConsumers.h:352, :358`) | **medium-high** — a forced synchronous grass rebuild destroys and re-creates instanced components | no |
| 6 | `Environment/LandscapeHandler.cpp:574` | `landscape.create` | `SpawnActor<ALandscape>` + `FScopedTransaction` + `ALandscapeProxy::Import` (builds and registers the whole `ULandscapeComponent` grid) | **high** | no |
| 7 | `Environment/LandscapeHandler.cpp:1122` | `landscape.sculpt` | `FLandscapeEditDataInterface::SetHeightData` + `Flush()` with `bUploadTextureChangesToGPU=true` | **medium** — GPU texture upload on the handler stack | no |
| 8 | `Environment/LandscapeHandler.cpp:1563` | `landscape.set_material` | `PostEditChangeProperty(LandscapeMaterial)` -> per-component render-state recreation + per-MIC `FlushRenderingCommands` | **high** — family I; named in `SafePoint.cpp:397` as needing exactly this fix | no, and `SafePoint.cpp` says a table entry would not work |
| 9 | `Environment/LandscapeHandler.cpp:1711` | `landscape.create_grass_type` | `CreatePackage` + `McpSafeAssetSave` | **medium** | no |
| 10 | `Environment/LandscapeHandler.cpp:1920` | `landscape.edit` | `FScopedSlowTask(2.0f)` + `SetHeightData` + GPU `Flush()` | **medium** — a slow task pumps Slate on its own stack | no |
| 11 | `Environment/LandscapeHandler.cpp:2876` | `landscape.create_procedural_terrain` | `FScopedSlowTask` + `FScopedTransaction` + weightmap allocation + layer paint (component material-permutation rebuild) | **high** | no |

**Nine of the eleven carry a real pump, and not one of them is in the table.** The two that do
not (`asset.validate`, `asset.set_tags`) still want the marshal gone, but as cleanup.

### Group 2 — marshal inside a job's bind delegate (4 sites) — a DIFFERENT case, do not lump

| file:line | verb | why it is not the same defect |
|---|---|---|
| `Blueprint/BlueprintApiIndexHandler.cpp:67` | `blueprint.build_api_index` | `FHandlerContext::StartJob` invokes `Args.BindNativeDelegate` **synchronously** (`HandlerContext.cpp:627`, cited at `SafePoint.h:138`). Deleting the marshal here would run the job body inline and break the ticket contract the verb advertises. |
| `Level/LevelHandler.cpp:381` | `level.save` | same |
| `Level/LevelHandler.cpp:488` | `level.save_as` | same |
| `Render/RenderHandler.cpp:1911` | `render.nanite_rebuild_mesh` | same — but its body is `UStaticMesh::Build()`, family H.4 again, so it wants `DeferToSafePoint` inside the bind delegate rather than deletion. |

These four are still outside the reentrancy guard and still land on the named-thread queue. They
need the *other* remedy, not this one; listed here so a fix pass does not delete them by pattern
match.

## Severity

Impact class **Critical** by the rubric (editor crash — the mechanism has two shipped kills on
record through a sibling verb, plus the `asset.reload` kill that cost six streams their state),
bumped down one for **reach** (landscape write verbs and two `asset.*` maintenance verbs are
regular but not every-session) -> **High**. No fault has been observed through any of these
eleven verbs specifically; the severity is the mechanism's, discounted for the missing direct
observation, and matches `B-compile-material-not-tick-gated` which is High on the same family
with faults observed.

## Fix shape

Three parts. Parts 1 and 2 are separable per verb; part 3 is what stops the regression.

1. **Delete the redundant marshal at all eleven Group-1 sites.** The body answers through `Ctx`
   directly, exactly as `asset.reload` now does — the whole call then runs inside
   `bProcessingRequest`, and an RPC arriving mid-operation is queued instead of executed
   underneath it. Where the body becomes synchronous the `FAsyncResponseToken` goes with it;
   `Ctx.SendSuccess`/`SendError` hit `FResponseCapture` on the same path
   (`B-landscape-handler-bypasses-ctx`, DONE, is what made the two equivalent). **Re-read, do
   not assume,** the three tests that ticket left behind —
   `landscape.create.AsyncRoutesThroughCapture`, `asset.set_tags.AsyncRoutesThroughCapture`,
   `AsyncTokenMigration.HandlersRegistered` — they pump the game-thread queue and assert
   `Capture.bWasCalled`; a synchronous body should still satisfy them, but that is a claim to
   verify, not to state.

2. **Gate the nine hazardous verbs.** Route choice is decided by `SafePoint.h:353-362` ("use the
   table unless a cross-dispatch caller can arrive on an ungated stack"), and here it decides
   against the table for most of them: `environment.build` forwards `create_landscape`,
   `paint_landscape`, `sculpt_landscape`, `modify_heightmap`, `set_landscape_material`,
   `create_landscape_grass_type`, `create_procedural_terrain` and `generate_lods` through
   `CrossDispatchEnv` -> `DispatchMethod` (`EnvironmentHandler.cpp:83-92, :389-420`), which
   bypasses `ProcessRequest` and therefore the table. So:
   - `landscape.create` / `edit` / `sculpt` / `set_material` / `create_grass_type` /
     `create_procedural_terrain` and `asset.generate_lods` -> in-handler
     `PinWrightSafePoint::RunAtSafePoint` (`SafePoint.h:452`), the `level.load` shape.
   - `asset.fixup_redirectors` and `landscape.flush_grass` have no cross-dispatch caller ->
     one-line table entries, cheaper and uniform.
   Each entry needs a test, because gating changes when the response is delivered.

3. **A source-scan ratchet, modelled on `Tests/Infra/TestIrAndNestedLoadGuard.cpp`.** A
   behavioural test cannot see this class: the defect is a *position*, and the failure it
   produces is an editor kill (an absence of signal, not a red). Scan
   `Source/*/Private/Handlers/**` for `AsyncTask(ENamedThreads::GameThread` and hold the count
   at the Group-2 total (4) with those four sites named, so a new marshal anywhere in the
   handler tree fails by count and a Group-1 removal that is later reverted fails by name. Run
   the input through `NeutralizeSourceText` (`Tests/TestUtils.h`) first — the two
   do-not-re-add comments at `AssetManageHandler.cpp:1801` and `LevelHandler.cpp:251` are
   prose about the pattern and must not score, which is board
   `B-error-code-adoption-test-scans-comments` exactly. Assert the regex works against a fixed
   sample so an ICU-less build cannot pass by matching nothing.

**Workaround:** none available to a caller. There is no in-band way to know which stack a
marshalled body will land on, and no parameter that changes it.

## History
- `#1-filed-the-follow-up-safepoint-promised` `OPEN` reporter — Filed from a source read alone (no editor launched, no repro attempted), as the follow-up `SafePoint.cpp:410` says is "filed separately" and that no ticket on this board turned out to carry. Confirmed the marshal is redundant at `RpcDispatcher.cpp:586` (game-thread hop) against `:696` (safe-point gate) and `:745-784` (`bProcessingRequest`), then classified all fifteen `AsyncTask(ENamedThreads::GameThread` sites under `Handlers/` by reading each deferred body. Eleven are the `asset.reload` shape; nine of those eleven pump the engine (render-state recreation, `UStaticMesh::Build`, package re-saves and deletes, `FScopedSlowTask`, GPU texture flush, forced grass regeneration) and **none** of the nine is in `GTickUnsafeMethodNames` (42 entries). The remaining four sit inside a job bind delegate, which `StartJob` invokes synchronously (`HandlerContext.cpp:627` via `SafePoint.h:138`), so for those the marshal is load-bearing and deleting it would break the ticket contract — they are listed separately so a pattern-match fix pass does not touch them. Two corrections to the finding that prompted this: `asset.validate` is NOT a hazard — `SafePoint.cpp:469-473` already adjudicates it harmless (it re-reads a resident `UObject`, no eviction, no fixup, no GC) and removing its marshal is cleanup, not a fix; and the "~17 sites" figure counts two comment lines, the real call count is 15. The named defect is `landscape.set_material` (`LandscapeHandler.cpp:1563` -> `:1600` -> `PostEditChangeProperty` at `:312`), which reaches hazard family I's engine code, whose shipped evidence is two editor kills on 2026-08-16, and which cannot be fixed by a table entry for two independent reasons — the marshal, and the `environment.build` cross-dispatch at `EnvironmentHandler.cpp:407`. Severity High: Critical impact class (editor crash) bumped down one for reach, no fault yet observed through these specific verbs.
