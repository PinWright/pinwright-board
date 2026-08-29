---
id: B-create-grass-type-addzeroed-never-renders
title: "landscape.create_grass_type builds its FGrassVariety with AddZeroed, bypassing the constructor that sets EndCullDistance and AllowedDensityRange — two independent engine gates then discard the variety, so the saved asset renders zero grass while the verb reports created plus a valid asset_path"
status: DONE
severity: High
category: bug
tags: [landscape, create_grass_type, grass, foliage, vegetation, addzeroed, constructor-bypass, silent-false-success, no-readback, cull-distance, persisted-artifact, one-line-fix]
---

# The asset is created, saved, and inert

`landscape.create_grass_type` allocates its one `FGrassVariety` with `AddZeroed`:

```cpp
int32 NewIndex = GrassType->GrassVarieties.AddZeroed();
FGrassVariety& Variety = GrassType->GrassVarieties[NewIndex];
```

`Handlers/Environment/LandscapeHandler.cpp:1572-1573`. `AddZeroed` memsets the new element and
**never runs `FGrassVariety::FGrassVariety()`**, which is where every non-zero default lives
(`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeGrass.cpp:1480-1511`). The handler
then writes seven fields by hand (`:1574-1580`: `GrassMesh`, `GrassDensity.Default`,
`ScaleX/Y/Z`, `RandomRotation`, `AlignToSurface`), saves with `McpSafeAssetSave` (`:1582`), and
answers `success: true` + `asset_path` (`:1583-1587`).

Everything the constructor would have set and the handler does not write is left at zero. Two of
those zeros are load-bearing.

## Gate 1 — `EndCullDistance == 0` discards the variety before it is ever considered

`LandscapeGrass.cpp:2998-2999`:

```cpp
int32 EndCullDistance = GrassVariety.GetEndCullDistance();
if (GrassVariety.GrassMesh && GrassVariety.GetDensity() > 0.0f && EndCullDistance > 0)
```

The constructor sets `EndCullDistance(10000)` and `EndCullDistanceQuality(10000)`
(`:1488-1489`). `AddZeroed` leaves both `0`, so `GetEndCullDistance()` (`:1530-1540`) returns `0`
down either branch and the `if` is never entered. No cluster is built, nothing is scattered, and
nothing anywhere logs a reason.

Note the density term in the same condition is a second, quality-level-dependent failure. The
handler writes `GrassDensity.Default` but not `GrassDensityQuality`, which the constructor sets
to `400.0f` (`:1483`). `GetDensity()` (`:1542-1552`) reads `GrassDensityQuality` whenever
`GEngine->UseGrassVarityPerQualityLevels` is on (`IsGrassQualityLevelEnable()`, `:1513-1516`), so
on such a host the density is `0` as well — the same `if` fails twice for two different reasons.

## Gate 2 — `AllowedDensityRange == (0,0)` is unsatisfiable

Both instance-placement loops apply the same weight window:

- Halton path — `LandscapeGrass.cpp:2414-2415`
- grid path — `LandscapeGrass.cpp:2477-2478`

```cpp
bKeep = (Weight > AllowedDensityRange.Min) && (Weight <= AllowedDensityRange.Max) && ...
```

The constructor sets `AllowedDensityRange(0.0f, 1.0f)` (`:1491`). Zeroed it is `(0, 0)`, so the
predicate reduces to `Weight > 0 && Weight <= 0` — false for every possible weight. Even if gate 1
were somehow passed, every candidate instance is rejected.

The two gates are independent: fixing either alone still yields zero grass.

## Collateral zeros, for completeness

`bUseGrid` is constructed `true` (`:1484`) and zeroed to `false`, silently switching the asset from
the grid placement path to the Halton path — a behavioural change no caller asked for and no
response field mentions. `PlacementJitter` 1.0 -> 0, `StartCullDistance` 10000 -> 0, `MinLOD` -1
(auto) -> 0 (forces LOD0), and `bAlignToTriangleNormals` / `bCastDynamicShadow` /
`bCastContactShadow` / `bReceivesDecals` all true -> false (`:1484-1503`). None of these is the
reason no grass renders; they are the reason the asset would still be subtly wrong after the two
gates were somehow satisfied.

`FGrassVariety` is also **not a POD**. It holds three `FPerQualityLevel*` members, each carrying a
`TMap<int32,int32>` plus the base's two `FString`s (`Runtime/Engine/Public/PerQualityLevelProperties.h:202`,
`:204`, `:240`, `:280`), and the constructor's body calls `SetQualityLevelCVarForCooking` on all
three (`LandscapeGrass.cpp:1508-1510`) — never run here, so the cook-time CVar bindings are empty
strings. UE's containers survive an all-zero state on destruction, so this does not crash; it is
construction bypass on a struct that has a meaningful constructor, which is the general defect.

## Why nothing caught it

Nothing exercises the verb past registration. The only test is
`PinWright.landscape.create_grass_type.ValidParamsNoCrash`
(`Tests/World/TestEnvironmentHandlers.cpp:1821-1833`): it posts
`{name: "TestGrassType", meshPath: "/Game/Meshes/SM_Grass"}` and asserts only that
`InvokeHandler` returned true — i.e. that a handler is registered for the method. It does not read
the response, does not check the asset exists, and `/Game/Meshes/SM_Grass` does not exist in this
project, so the handler's own `ASSET_NOT_FOUND` path is what the test actually exercises. This is
the "a missing required fixture is a test FAILURE, never a skip" rule from
`agent-conventions.md` being violated in the weakest possible direction: the test greens without
covering anything. Coverage is tracked separately in `F-foliage-namespace-has-no-behavioural-tests`.

There is also no authored `ULandscapeGrassType` anywhere in the host project — the only one present
is Epic's stock `LGT_Grass` — so the verb has never been run against a real mesh by anyone.

## Controls checked, so the fix stays narrow

- **The other `AddZeroed` call sites in the plugin are correct.**
  `AudioGen/PwAudioFeatures.cpp:664`, `:666` and `AudioGen/PwFxChainB.cpp:655-656` are
  `Audio::FAlignedFloatBuffer` (a `TArray<float>`) sized to zero — zero *is* the intended value.
  The remaining hits (`Tests/Assets/TestPwMusicGraph.cpp:98`,
  `Tests/Media/TestPwDecomposeTracking.cpp:757-758`) are the same POD-buffer shape in tests.
  `Handlers/Actor/SpawnMaterialUtils.h:355` and `Handlers/Audio/AudioAuthoringHandler.cpp:2440`
  are comments *about* engine `AddZeroed` behaviour, not call sites.
- **`foliage.add_type` is unaffected.** It builds its type with
  `NewObject<UFoliageType_InstancedStaticMesh>` (`Handlers/Environment/FoliageHandler.cpp:601`),
  so the CDO defaults are intact; it writes `Density`, `ScaleX/Y/Z`, `AlignToNormal`,
  `ReapplyDensity` on top (`:610-620`). Same domain, same authoring shape, no defect — which is
  what makes `AddZeroed` here look like a local slip rather than a house pattern.
- **`AddDefaulted` links in this DLL.** `Tests/Assets/TestNewTypeDumpBuilders.cpp:365-374` already
  uses `GrassVarieties.AddDefaulted_GetRef()` on UE >= 5.4 and falls back to `AddZeroed` only
  below that, because `FGrassVariety`'s constructor was not exported before 5.4. On UE 5.8 the
  export is present — `UE_API FGrassVariety();` with `#define UE_API LANDSCAPE_API`
  (`Runtime/Landscape/Classes/LandscapeGrassType.h:14`, `:35`) — so the fix needs no new plumbing.
  That test file's comment, *"AddZeroed is the same construction the production create path already
  uses (LandscapeHandler.cpp)"*, is a live cross-reference to this defect written by someone who
  did not realise it was one.

## Fix

Replace `AddZeroed` with `AddDefaulted` at `LandscapeHandler.cpp:1572`:

```cpp
int32 NewIndex = GrassType->GrassVarieties.AddDefaulted();
```

If the plugin must still build below UE 5.4, mirror the
`UE_VERSION_NEWER_THAN_OR_EQUAL(5, 4, 0)` guard `TestNewTypeDumpBuilders.cpp:365` already carries,
and on the pre-5.4 branch write `EndCullDistance`, `EndCullDistanceQuality`, `AllowedDensityRange`
and `bUseGrid` explicitly rather than leaving them zeroed.

Two things belong with the fix, not after it:

1. **A readback.** Even correctly constructed, this verb reports only `success` and `asset_path`.
   Echo the resolved `endCullDistance`, `density` and `allowedDensityRange` from the variety after
   construction, so the response carries the numbers the engine gates on rather than only the
   numbers the caller supplied. Without this, the *next* zeroed field is as invisible as this one
   was.
2. **A behavioural test.** Create a grass type from a real mesh, then assert on the saved asset that
   `GrassVarieties[0].GetEndCullDistance() > 0` and `AllowedDensityRange.Max > 0`. It must
   demonstrably fail against the pre-fix code (per `agent-conventions.md` on differential proof),
   which it does: both read 0 today.

**Suggestion for `docs/lessons.md`** (recorded here so it is not lost; that file is out of scope for
this ticket): *`AddZeroed` on a `USTRUCT` with a meaningful constructor writes a silently-invalid
record — the fields you set look right in the details panel and every field you did not set is a
plausible-looking zero.* This is a distinct mechanism from the existing "the field is not where the
behaviour is" entry: here the field **is** where the behaviour is, and it was never written.

## Not RPC-verified

Everything above is source-read. The editor was not running for this pass, so no
`landscape.create_grass_type` call was made and no grass was rendered. An editor test would settle
one thing this analysis cannot: whether a landscape whose material declares a grass output would
render grass from a variety built with `AddDefaulted` and the handler's seven writes, i.e. whether
these two gates are the *only* ones. They are sufficient to prove the current asset is inert;
proving the fixed asset is live needs a render.

## Same shape as

`B-mrq-render-result-omits-bitrate-and-size`, `B-ground-probe-hits-hull-not-render`,
`B-niagara-validate-green-while-component-inactive` — the call succeeds, every number it reports is
correct, and the output is wrong because the deciding number was never reported. Here the deciding
numbers are `EndCullDistance` and `AllowedDensityRange`, and the artifact is persisted to disk and
committed, so the lie outlives the session.

Closest structural sibling is `B-ortho-capture-culls-distant-foliage` (OPEN, High): both are *an
engine cull distance silently zeroing the visible result*, and that ticket walks the
`EndCullDistance * MaxDrawDistanceScale` path directly. Different cause — a scalability CVar there,
an unconstructed field here — but a fixer holding both sees the class.

severity rationale: impact=High — silent false-success writing a persisted artifact; the verb reports `created` with a valid `asset_path`, the asset opens with populated-looking fields, and every zero the engine gates on reads as a deliberate zero × reach=normal — `landscape.create_grass_type` is one of six `landscape` verbs and the only grass one, not a rare edge path, so no reach modifier applies (the fact that this project has authored zero grass types is an encounters observation, which the rubric excludes as a severity input) -> High

## History
- `#1-addzeroed-bypasses-constructor` `OPEN` reporter — Source-read only, editor not running; nothing here is RPC-verified. `LandscapeHandler.cpp:1572` uses `GrassVarieties.AddZeroed()`, bypassing `FGrassVariety::FGrassVariety()` (`LandscapeGrass.cpp:1480-1511`). Two independent engine gates then discard the variety: `:2999` requires `EndCullDistance > 0` against a constructor default of 10000 (`:1488-1489`), and `:2414` / `:2477` require `Weight > AllowedDensityRange.Min && Weight <= Max` against a constructor default of `(0,1)` (`:1491`) — zeroed to `(0,0)` the predicate is unsatisfiable for every weight. Density fails a third time on hosts with `UseGrassVarityPerQualityLevels`, because the handler writes `GrassDensity.Default` but not `GrassDensityQuality` (ctor 400.0f, `:1482`; read by `GetDensity()` `:1542-1552`). Fix is `AddZeroed` -> `AddDefaulted`; `AddDefaulted_GetRef` is already proven to link in this DLL on UE >= 5.4 by `Tests/Assets/TestNewTypeDumpBuilders.cpp:365-374`, and `FGrassVariety` is exported on 5.8 (`LandscapeGrassType.h:14`, `:35`). Dedup: searched the board for `AddZeroed`, `GrassVariety`, `grass_type`, `create_grass_type` and `GrassType` — seven files match, none is this defect. `B-landscape-handler-bypasses-ctx` (DONE) touches the same handler and even names `create_grass_type` in its `#6` entry, but is about async response routing through `FHandlerContext`, not about what the lambda writes. `B-create-procedural-terrain-paints-nothing` (DONE) is the sibling silent-false-success on `landscape.create_procedural_terrain`, different verb and different cause. `F-landscape-height-readback`, `B-environment-build-dispatcher-rejects-forwarded-params` and the four `asset-dump` / `asset-list` hits matched only on incidental prose. No ticket covers `landscape.create_grass_type` correctness. Controls checked so the fix stays narrow: the other five `AddZeroed` sites in the plugin are `TArray<float>` audio buffers where zero is the intended value (`PwAudioFeatures.cpp:664`, `:666`; `PwFxChainB.cpp:655-656`; plus two test files), and `foliage.add_type` builds via `NewObject` (`FoliageHandler.cpp:601`) and is unaffected. Survived because nothing exercises the verb: `PinWright.landscape.create_grass_type.ValidParamsNoCrash` (`Tests/World/TestEnvironmentHandlers.cpp:1821-1833`) asserts only that a handler is registered, against a `meshPath` that does not exist in this project. Also recorded here: a `docs/lessons.md` line worth adding (that file deliberately not edited by this ticket) — `AddZeroed` on a `USTRUCT` with a meaningful constructor writes a silently-invalid record.
- `#2-default-construct-the-variety` `IN-REVIEW` developer — "Core defect confirmed in source, mechanism intact. `LandscapeHandler.cpp` (now `:1578`, ticket line numbers stale) built its one `FGrassVariety` with `AddZeroed`, so `FGrassVariety::FGrassVariety()` (`LandscapeGrass.cpp:1480-1511`) never ran: `EndCullDistance`/`EndCullDistanceQuality` 10000 -> 0 (scatter loop at `:2998-2999` requires `GetEndCullDistance() > 0`), `AllowedDensityRange` (0,1) -> (0,0) (placement windows at `:2414` / `:2477` become `Weight > 0 && Weight <= 0`, unsatisfiable), `GrassDensityQuality` 400 -> 0, `bUseGrid` true -> false. Fixed by default-constructing the variety instead of populating fields by hand: `AddDefaulted_GetRef()` on UE >= 5.4, and on 5.3 — where `FGrassVariety` carries no `LANDSCAPE_API` (`LandscapeGrassType.h:29`, vs `struct LANDSCAPE_API FGrassVariety` at the same line on 5.4) so the ctor is not exported and `AddDefaulted_GetRef` would not link — `FGrassVariety::StaticStruct()->InitializeStruct()`, which runs that same constructor through the Landscape module's own struct ops; the generated `StaticStruct()` IS `LANDSCAPE_API` on every supported version, verified in 5.3's `LandscapeGrassType.generated.h:19`. This deliberately avoids the ticket's suggested pre-5.4 fallback of hand-writing the defaults — see correction (2). Also now writes the caller's density to `GrassDensityQuality` as well as `GrassDensity`, because `GetDensity()` (`:1542-1552`) reads the per-quality slot on hosts with `GEngine->UseGrassVarityPerQualityLevels` and would otherwise have kept the constructor's 400 there. Readback added per the ticket: the response now carries `density` and `end_cull_distance` read back off the stored variety (both from `.Default`, which is honest precisely because the per-platform and per-quality slots are kept in agreement). `AllowedDensityRange` is deliberately NOT echoed — it does not exist before 5.5, and a version-conditional response field would make every consumer version-conditional too. Files: `Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp`. Test: `PinWright.landscape.create_grass_type.VarietyIsRenderable` in `Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp` — drives the real handler (shared capture + `PumpUntilCaptured`; the verb completes inside an `AsyncTask(GameThread)`) with `/Engine/BasicShapes/Cube.Cube` at density 250, finds the created `ULandscapeGrassType` in memory and asserts on `GrassVarieties[0]`: `GrassMesh` non-null, `EndCullDistance.Default > 0`, `EndCullDistanceQuality.Default > 0`, both density slots == 250, `AllowedDensityRange.Max > 0` (guarded 5.5+), `bUseGrid` true, plus the two echoed response fields. Differential: against the `AddZeroed` code every one of those reads 0/false while the handler still reports success, so the test fails pre-fix. A missing fixture is a hard failure, not a skip; the created package is discarded on scope exit (flags cleared, `PackageDeleted`, dirty flag cleared, marked garbage). Not compiled or run — the orchestrator owns builds. TWO CORRECTIONS TO THE TICKET. (1) `McpSafeAssetSave` does NOT write to disk: its entire body is `MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()` (`Utils/AssetUtils.cpp:362-377`), with a comment saying immediate save was removed on 5.7+ to avoid bulkdata corruption. The asset is an in-memory dirty package until a human runs Save All, so 'the saved asset', 'persisted artifact' and 'the artifact is persisted to disk and committed, so the lie outlives the session' overstate the blast radius — absent a manual save the lie dies with the editor session. Severity stands on the silent false-success, not on persistence; the `persisted-artifact` tag is inaccurate. (2) The ticket's pre-5.4 fallback advice — write `EndCullDistance`, `EndCullDistanceQuality`, `AllowedDensityRange` and `bUseGrid` explicitly — would not compile on the versions it targets: `AllowedDensityRange` arrived in UE 5.5 (absent from `LandscapeGrassType.h` on 5.3 and 5.4, present 5.5-5.8), and `bAlignToTriangleNormals` is likewise 5.5+. Hand-copying the constructor's defaults per engine version is exactly the drift this fix avoids. Everything else in the ticket verified as written: the constructor's values, both gate sites, the 5.3-vs-5.4 export difference, and that `PinWright.landscape.create_grass_type.ValidParamsNoCrash` reaches only the `ASSET_NOT_FOUND` path because `/Game/Meshes/SM_Grass` does not exist in this host. That weak test is left in place; coverage is tracked in `F-foliage-namespace-has-no-behavioural-tests`. No overlap with the four in-flight foliage/scatter tickets — this touches only `landscape.create_grass_type` and its test, no `foliage.*` verb and no `spatial.*` file. Still not RPC-verified: the ticket's open question, whether a landscape whose material declares a grass output actually renders from a correctly-constructed variety (i.e. whether these two gates are the only ones), needs an editor render and remains open."
- `#3-verified-variety-is-renderable-on-disk` `DONE` tester — **Verified live, and the disk half is verified separately from the memory half.** A `ULandscapeGrassType` created by the fixed verb dumps with **nothing zeroed**: `GrassDensity.Default` **300** *and* `GrassDensityQuality.Default` **300**; `StartCullDistance.Default` / `EndCullDistance.Default` both **10000** *and* `StartCullDistanceQuality.Default` / `EndCullDistanceQuality.Default` both **10000**; scale range **0.70-1.15**; `AlignToSurface`, `RandomRotation` and `bUseGrid` all **true**. Every field `#2` names as constructor-supplied is present at its constructor value, and the two gates the body names are both satisfied (`EndCullDistance > 0`; `AllowedDensityRange` non-degenerate). **The `*Quality` half is called out deliberately, because it is `#2`'s own addition and the half that would have regressed silently.** `grep -a` over the saved `.uasset` bytes finds all three of `GrassDensityQuality`, `StartCullDistanceQuality` and `EndCullDistanceQuality` — re-derived by this tester rather than taken on report, over all four grass types under `Content/VegetationTest/GrassTypes/` (`LGT_VegTest_Meadow`, `…_MeadowBase`, `…_MeadowFleck`, `…_MeadowTuft`): each file's bytes carry all three names. So the fix is **on disk**, not only in the loaded object — which is the only form of this check that can fail, per this project's rule that a read-back through `load_asset` returns the in-memory object and never consults the file. On hosts with `GEngine->UseGrassVarityPerQualityLevels` the per-quality slot is the one `GetDensity()` reads, so a fix that lived only in `GrassDensity` would have rendered nothing there while every readback looked correct. Source re-derived at HEAD, `#2`'s line numbers having moved again: `Handlers/Environment/LandscapeHandler.cpp:1587` is `GrassVarieties.AddDefaulted_GetRef()` under the UE >= 5.4 branch, `:1593-1594` the pre-5.4 `AddZeroed()` + `FGrassVariety::StaticStruct()->InitializeStruct(&Variety)` fallback, and `:1601` the `Variety.GrassDensityQuality.Default` write; the rationale comment sits at `:1578-1585`. **The end-to-end question `#2` left open is answered, in the affirmative, for this configuration.** `#2` closed saying it was still unverified "whether a landscape whose material declares a grass output actually renders from a correctly-constructed variety (i.e. whether these two gates are the only ones)". It does: grass renders through a `LandscapeGrassOutput` on a real landscape material, captured at `Saved/Screenshots/OpenLevel/za_13_zone_overview.png`. Answer is scoped to this host and this material — it establishes that no THIRD undiscovered gate blocks a correctly-constructed variety here, not that none exists anywhere. **What this close does NOT cover, stated rather than absorbed.** Three further defects were found live in the same pass and are being filed as their own tickets, deliberately not held against this one, because the defect this ticket describes (a zero-initialised variety the engine discards) is gone: (1) `E-create-grass-type-cull-distance-and-output-path` — the verb writes `StartCullDistance == EndCullDistance`, so grass pops in with no fade band, and takes no cull-distance parameters at all; (2) `B-grass-varieties-edit-does-not-reach-renderer` — editing `GrassVarieties` and saving does not reach the renderer without `system.console_command {command:"grass.FlushCache"}`; (3) `F-landscape-set-grass-output` — no verb wires a created grass type into a material's `MaterialExpressionLandscapeGrassOutput`, so the verb can produce a perfectly valid asset that nothing PinWright is able to build will ever render. Recorded honestly: at the time this entry was written none of those three files existed on the board, so the ids are forward references to tickets being filed in this same session, not links a reader can follow yet. `#2`'s two corrections to the body stand unchallenged and are not re-litigated here: `McpSafeAssetSave` does not write to disk (the asset used above was saved by other means), and the `persisted-artifact` tag remains inaccurate.
