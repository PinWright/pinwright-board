---
id: B-foliage-writes-vetoed-by-scalability-cvars
title: "sg.FoliageQuality drives grass.densityScale and foliage.DensityScale, so landscape.create_grass_type echoes a density the grass builder will not use and the foliage counters report instances the renderer will not draw — no verb reads either cvar"
status: IN-REVIEW
severity: High
category: bug
tags: [foliage, landscape, grass, scalability, cvar, measured-vs-requested, silent-noop, misleading-success, audit, sweep-extension]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# `sg.FoliageQuality` is the next member of the scalability-veto class, and it is on nobody's sweep list

`sg.FoliageQuality` was `1` at editor start on this host. `[FoliageQuality@1]` sets
`foliage.DensityScale=0.4` and `grass.DensityScale=0.4`
(`C:/UE_5.8/Engine/Config/BaseScalability.ini:972-974`); `[FoliageQuality@0]` sets both to **`0`**
(`:964-966`). `@2` is 0.8 (`:980-982`), `@3` and `@Cine` are 1.0 (`:988-990`, `:996-998`).

`grep -rn "DensityScale" Plugins/PinWright/Source` returns **zero** hits at this checkout's HEAD.
`sg.FoliageQuality` appears exactly once, as a row in `performance.set_scalability`'s read-back list
(`Handlers/Debug/PerformanceHandler.cpp:425`). No foliage or landscape verb reads either cvar, and
none reports one.

This ticket extends the sweep `B-lighting-writes-vetoed-by-scalability-cvars` (IN-REVIEW) opened.
That ticket's "same shape outside `lighting.*`" list names `r.DistanceFieldShadowing`,
`r.CapsuleShadows` and `r.LightFunctionQuality` — **`sg.FoliageQuality` / `foliage.*` is not on it**,
verified against its current text. So this is a new member of a known class, not a re-file.

## Two different mechanisms, and they are not interchangeable — traced to engine source

The premise "one cvar scales every foliage number" does not survive reading the engine. There are
two cvars under one scalability group and they act at different stages:

**`grass.densityScale` — a true density multiplier, and it needs no opt-in.**
Registered at `Runtime/Landscape/Private/LandscapeGrass.cpp:131-135` (`ECVF_Scalability`, note the
registered name is lower-case `densityScale`). Applied at `:2040-2041`:

```cpp
const float DensityScale = bEnableDensityScaling ? GGrassDensityScale : 1.0f;
GrassDensity = GrassVariety.GetDensity() * DensityScale;
```

The per-type opt-out is `ULandscapeGrassType::bEnableDensityScaling`
(`Runtime/Landscape/Classes/LandscapeGrassType.h:176-183`), and its constructor sets it **`true`**
(`LandscapeGrass.cpp:1567`). So a grass type created with engine defaults **is** scaled, always.

**`foliage.DensityScale` — a render-time cull, and it is opt-in and OFF by default.**
Registered at `Runtime/Engine/Private/HierarchicalInstancedStaticMesh.cpp:135-140`
(`ECVF_Scalability`), help text verbatim: *"Controls the amount of foliage to render. Foliage must
opt-in to density scaling through the foliage type."* It feeds `CurrentDensityScaling` (`:3071-3076`,
and the cvar sink at `:176-196`), which is handed to `FClusterBuilder` (`:2675`, `:2891`) — the HISM
render cluster tree. Instances stay in `PerInstanceSMData`; a fraction are excluded from the tree and
therefore from the frame. `:3096` special-cases `CurrentDensityScaling == 0.f`.
The gate is `UFoliageType::bEnableDensityScaling` (`Runtime/Foliage/Public/FoliageType.h:583-591`),
propagated to the component at `Runtime/Foliage/Private/InstancedFoliage.cpp:1750-1752` and to the
ISM descriptor at `FoliageISMActor.cpp:92` — and `UFoliageType`'s constructor sets it **`false`**
(`InstancedFoliage.cpp:669`). In the editor there is a second gate, `bCanEnableDensityScaling`
(`HierarchicalInstancedStaticMesh.cpp:190`, `:3071`).

Consequence for scoping this ticket honestly: **the grass half fires on default assets, the foliage
half fires only on a foliage type somebody opted in.** Both are worth reporting, for different
reasons — see below.

## What each verb publishes today

| verb | field | file:line | what the cvar does to it |
|---|---|---|---|
| `landscape.create_grass_type` | `density` | `Handlers/Environment/LandscapeHandler.cpp:1616` | echoed from `Variety.GrassDensity.Default` written at `:1600-1601`; the builder will use `density * grass.densityScale`. At `sg.FoliageQuality 1` the echo is **2.5x** the effective density; at `@0` the effective density is **0**. |
| `foliage.paint` | `instancesPlaced` | `Handlers/Environment/FoliageHandler.cpp:546` | truthful count of stored instances; on an opted-in type the renderer draws `instancesPlaced * foliage.DensityScale` of them, **zero** at `@0`. |
| `foliage.add_instances` | `instancesPlaced` | same writer, `FoliageHandler.cpp:546` | same. |
| `foliage.get_instances` | `count` | `FoliageHandler.cpp:832` | reads storage, not the frame; same divergence. |
| `foliage.remove` | `instancesRemoved` | `FoliageHandler.cpp:689` | storage-truthful, same divergence in what it implies about the frame. |
| `foliage.add_type` | writes `Density` | `FoliageHandler.cpp:956` (+ `ReapplyDensity` `:959`) | never writes or reports `bEnableDensityScaling`, so the caller cannot tell which regime the type is in. |
| `foliage.create_procedural` | per-type `density` | `FoliageHandler.cpp:1447` | same. |
| auto-created fallback types | `Density = 100.0f` | `FoliageHandler.cpp:389`, `:1195` | created with `bEnableDensityScaling` at its `false` default; the caller is never told. |

`landscape.create_grass_type` is the sharpest case and the one that matches the precedent exactly: it
**echoes a number it just wrote** as if it were the effective one. That is the `spawn_sky_light`
scaled-intensity shape named in `B-lighting-writes-vetoed-by-scalability-cvars` — *"a cvar that does
not disable the write but changes what it means"* — one namespace over and unlisted.

The foliage counters are the `setup_volumetric_fog` shape instead: the number is literally true and
implies something false about the frame. At `sg.FoliageQuality 0` a verb can report placing N
instances of which **zero** render.

## Why this is a defect and not a host misconfiguration

The board has already settled this, twice, and the ticket is built on that rather than on a fresh
argument.

`B-setup-volumetric-fog-enabled-true-while-cvar-off` (DONE, High, `encounters: 4`) records in its
`#1` that the project-config workaround was applied **and the ticket stayed open anyway**:

> Worked around by pinning `r.VolumetricFog=1` under `[SystemSettings]` in the project's
> `DefaultEngine.ini`; re-confirmed still `1` after a machine restart this session via
> `system.console.search`. […] **The plugin defect is untouched:** the verb still answers
> `enabled: true` when volumetric fog cannot render.

That ticket also states the class this one joins, under *"Generalise: every verb a scalability cvar
can veto"*: **"This is one instance of a class."**

`B-light-shaft-flags-decorative-under-scalability` (DONE, High) states the rule the fog fix
established:

> **any component flag whose renderer pass a scalability cvar can switch off has this failure mode.**

## The ask, fixed by precedent

`B-showflag-cvar-override-contaminates-capture` (DONE, High) fixes the shape of the remedy. Its `#1`:

> **Reported, never refused**: a shared editor condition the caller may knowingly accept must not
> fail the capture.

and the companion sentence, which is in `B-setup-volumetric-fog-enabled-true-while-cvar-off` `#4`
(not in the showflag ticket — see the citation note at the end of this section):

> The verb deliberately does NOT set the cvar (the ticket's "Better" option): **flipping a
> scalability cvar permanently is global state a verb must not change silently, and reporting is the
> treatment that generalises** to the sibling verbs listed under "Generalise".

So:

1. **Measure**, through `IConsoleManager::Get().FindConsoleVariable`, at the point each handler
   already assembles its response. `system.console.search` already proves the registry read works
   live and returns `currentValue`.
2. **Report requested vs effective.** `landscape.create_grass_type` should publish the written
   `density` and an `effectiveDensity` beside it, plus
   `grassDensityScaleCVar {cvar, found, value}` and the grass type's own
   `bEnableDensityScaling` (which decides whether the scale applies at all). The foliage counters
   should publish `foliageDensityScaleCVar {cvar, found, value}` and the resolved
   `densityScalingEnabled` for the target type, so `instancesPlaced` stops being read as
   "instances in the frame".
3. **Warn, naming the remedy** — `system.console_command "sg.FoliageQuality 3"`, or a
   `[SystemSettings]` pin in `DefaultEngine.ini` — only when the measurement says the write is
   scaled or zeroed.
4. **Do not set the cvar.**
5. `foliage.add_type` / `foliage.create_procedural` should also accept and report
   `bEnableDensityScaling`. Today the flag is unreachable through the surface and unreported, so a
   caller cannot even opt in to the behaviour the engine documents, let alone know which regime a
   type is in.

**Fix sites**: `Handlers/Environment/LandscapeHandler.cpp` (the `landscape.create_grass_type`
response at `:1611-1619`) and `Handlers/Environment/FoliageHandler.cpp` (`:546`, `:689`, `:832`,
`:956`, `:1447`).

## Host conditions — evidence, not the defect

Recorded the way the fog ticket carries its `[SystemSettings]` pin: these are the conditions the
finding was made under, and they are **not** what makes it a defect.

- This project has **no `Config/DefaultScalability.ini`** — verified by listing
  `X:/src/unreal/EAContentExamples58/Config/`, which holds `DefaultEngine.ini`,
  `DefaultEditor.ini`, `DefaultGame.ini`, `DefaultDeviceProfiles.ini`, `DefaultInput.ini`,
  `DefaultPlugins.ini`, `DefaultGameplayTags.ini`, `DefaultMass.ini`,
  `DefaultEditorPerProjectUserSettings.ini` and the platform subfolders, and nothing else. So
  `BaseScalability.ini` governs the `sg.*` groups unopposed.
- The project's `DefaultEngine.ini` `[SystemSettings]` block pins `r.VolumetricFog=1`,
  `r.LightShaftQuality`, `r.LumenScene.SurfaceCache.AtlasSize=8192` and friends — the accumulated
  scar tissue from the two precedent tickets — but pins **nothing** in the foliage family.
- The machine-global `EditorSettings.ini` sits at Low for every group, which is how
  `sg.FoliageQuality` was 1 at editor start.
- Raised to `3` for the session and confirmed in the editor log. **Not durable** — it dies with the
  editor, like every cvar the precedent tickets touched.

`E-scalability-console-escape-hatch-misleading` (OPEN, Low) already draws this line for the board:

> Note: the ini pin that forced the pin-collision is partly this fuzz host's `DefaultEngine.ini`
> artifact, **but device profiles and project config pin `sg.*` on real projects too**, and the docs
> inaccuracy (the `scalability N` priority claim) is host-independent.

The same reasoning transfers: `sg.FoliageQuality` is pinned by device profiles on every shipping
platform, so a verb that reports an unscaled density is wrong on real projects, not just on a fuzz
host at Low.

That ticket is also worth reading before fixing, for a second reason: it establishes that the
aggregate `scalability N` console command writes at `ECVF_SetByScalability`, the lowest settable
priority, so it is **not** a reliable way for a fix's test to force `sg.FoliageQuality` — a per-cvar
`sg.FoliageQuality N` (ECVF_SetByConsole) is.

## Discovery cost, and why it belongs on this board

The scaling was invisible through the RPC surface at the moment it mattered. No `foliage.*` response
mentions a cvar; `system.console.search` can read `foliage.DensityScale` but only if you already
suspect it exists, and there is no read-back on the write side at all
(`F-console-batch-get-cvar-values`, OPEN, `encounters: 5` — this session's `#5` records the same
foliage work as evidence for its value-echo half). The route that actually worked was running the
bare cvar name in the console and grepping the editor log. A whole session's foliage work was tuned
against a frame the responses described inaccurately.

## Severity

Argued against the rubric rather than inherited from the precedents.

**Impact = High.** The rubric's High is *"silent wrong / stale / hardcoded data on a normal path (the
caller trusts a result that is a lie and builds on it)"*. `landscape.create_grass_type`'s `density`
echo is exactly that on a default-constructed grass type: `bEnableDensityScaling` defaults `true`, so
the reported number is 2.5x the effective one at `@1` and infinitely wrong at `@0`, and the caller
tunes against it. The foliage counters sit at the boundary of High and Medium — they are literally
true about storage — but their whole purpose is to tell the caller what is now in the level, and at
`@0` on an opted-in type they report N over an empty frame, which is the `setup_volumetric_fog`
`enabled: true` failure verbatim.

**Reach modifier: declined, in both directions, and named.** Bumping *down* to Medium would require
`foliage.*` / `landscape.create_grass_type` to be *"a rare edge path"*. They are not — they are the
only route to vegetation on any level that has any, and this project's whole environment pass runs
through them. Bumping *up* to Critical is not available: the rubric reserves Critical for a crash or
for data corruption, and nothing here corrupts an asset — the instances and the density value are
stored correctly, they are merely described in terms that do not match what the renderer will do.

**Net: High** — the same band as the two DONE precedents it extends, reached independently. It is
deliberately *not* filed at the Medium of `B-lighting-writes-vetoed-by-scalability-cvars`: that
ticket is Medium as a batch of already-understood follow-ons behind a landed pattern, whereas this is
a first sighting in a namespace nobody has audited, where the cvar names appear nowhere in plugin
source.

## Same shape as

- `B-setup-volumetric-fog-enabled-true-while-cvar-off` (DONE, High) — the pattern, the fix style, and
  the precedent that a project-config pin does not close a ticket of this class.
- `B-light-shaft-flags-decorative-under-scalability` (DONE, High) — the second confirmed member, and
  the statement of the general rule.
- `B-lighting-writes-vetoed-by-scalability-cvars` (IN-REVIEW, Medium) — the sweep this extends;
  `spawn_sky_light`'s scaled intensity is the closest analogue to the grass-density echo.
- `B-showflag-cvar-override-contaminates-capture` (DONE, High) — "reported, never refused", and the
  landed survey machinery in `Handlers/Render/PreviewViewportCaptureUtils.cpp` a foliage fix can
  copy.
- `B-set-scalability-no-sg-update` — the `sg.*` read-back half, orthogonal.

## Related

- `F-console-batch-get-cvar-values` (OPEN, Low) — why the veto was unobservable at the moment it
  mattered; this session's evidence is recorded there as `#5`.
- `E-scalability-console-escape-hatch-misleading` (OPEN, Low) — host-vs-real-project framing, and the
  `ECVF_SetByScalability` trap a fix's test must avoid.

**Citation note for whoever works this**: the "flipping a scalability cvar permanently is global
state a verb must not change silently" sentence lives in
`B-setup-volumetric-fog-enabled-true-while-cvar-off` history `#4`, not in
`B-showflag-cvar-override-contaminates-capture`. The showflag ticket's contribution is
*"Reported, never refused"* (`#1`). Both are quoted above from their real homes.

## History
- `#1-foliage-quality-drives-two-density-cvars` `OPEN` reporter — Found on host project EAContentExamples58 while building foliage; `sg.FoliageQuality` was 1 at editor start, raised to 3 for the session and confirmed in the editor log (not durable). Traced to `BaseScalability.ini:964-998`: `@0` sets `foliage.DensityScale=0` and `grass.DensityScale=0` (`:965-966`), `@1` sets both to 0.4 (`:973-974`). `grep -rn "DensityScale" Plugins/PinWright/Source` returns zero hits at HEAD; `sg.FoliageQuality` appears only as a read-back row in `PerformanceHandler.cpp:425`. **The two cvars act differently and the ticket is scoped to that rather than to the assumption they behave alike.** `grass.densityScale` (`LandscapeGrass.cpp:131-135`) multiplies grass density at build time (`:2040-2041`) and its per-type opt-out `ULandscapeGrassType::bEnableDensityScaling` defaults **true** (`:1567`), so `landscape.create_grass_type`'s echoed `density` (`LandscapeHandler.cpp:1616`, written `:1600-1601`) is 2.5x the effective value at `@1` and meaningless at `@0` on a default asset — the `spawn_sky_light` scaled-intensity shape from `B-lighting-writes-vetoed-by-scalability-cvars`, one namespace over and absent from that ticket's "same shape outside `lighting.*`" list (verified). `foliage.DensityScale` (`HierarchicalInstancedStaticMesh.cpp:135-140`, help text: "Foliage must opt-in to density scaling through the foliage type") instead culls at render time through `CurrentDensityScaling` → `FClusterBuilder` (`:3071-3076`, `:2675`, `:2891`, zero special-cased at `:3096`), and its gate `UFoliageType::bEnableDensityScaling` defaults **false** (`InstancedFoliage.cpp:669`) — so the foliage counters (`FoliageHandler.cpp:546`, `:689`, `:832`) diverge from the frame only on an opted-in type, and PinWright's auto-created types (`:389`, `:1195`) never opt in and never say so. At `sg.FoliageQuality 0` an opted-in type yields `instancesPlaced: N` over zero rendered instances. Ask, fixed by the precedents rather than invented: measure both cvars through `IConsoleManager`, report requested-vs-effective plus the type's `bEnableDensityScaling`, warn naming the remedy, and do NOT set the cvar. Host conditions recorded as evidence only: no `Config/DefaultScalability.ini` exists in this project (verified by listing `Config/`), so `BaseScalability.ini` governs unopposed, and `DefaultEngine.ini` `[SystemSettings]` pins the fog and light-shaft cvars from the precedent tickets but nothing in the foliage family. Severity High by impact (silent wrong data on a normal path, `landscape.create_grass_type`'s density echo), reach modifier explicitly declined in both directions.
- `#2-measured-density-published-beside-requested` `IN-REVIEW` developer — Adopted the `B-lighting-writes-vetoed-by-scalability-cvars` measured-vs-requested pattern for both halves of the `sg.FoliageQuality` group. New shared header `Handlers/Environment/ScalabilityDensityCVars.h` (`PinWrightDensityScalability`) holds the one cvar read, the `{cvar, found, value}` JSON builder, the two *different* scale resolvers — `grass.densityScale` is applied raw at `LandscapeGrass.cpp:2040-2041`, `foliage.DensityScale` is clamped to [0,1] at `HierarchicalInstancedStaticMesh.cpp:3076` before `FClusterBuilder` — and the shared foliage report block; the reference implementation in `LightingHandler.cpp` has no extractable helpers (its five cvar reads are open-coded), so nothing was lifted out of it and it was left untouched rather than refactored under three other agents editing the same wave. `landscape.create_grass_type` now publishes `densityScalingEnabled` (read off `ULandscapeGrassType::bEnableDensityScaling`, which defaults **true**, so a default-created grass type IS scaled), `grassDensityScaleCVar`, and `effectiveDensity` beside the stored `density`, with a `cvarWarning` naming `sg.FoliageQuality 3` / the `[SystemSettings]` pin when the two differ. `foliage.paint`, `foliage.add_instances` and `foliage.get_instances` publish `foliageDensityScaleCVar`, `densityScalingEnabled` and `expectedDrawnInstances` — the ledger count and what the renderer is expected to draw are now separate numbers instead of one that means neither; `get_instances` accumulates the expectation **per foliage type** because the opt-in is per type, and an orphaned info (null type, opt-in unknowable) marks the scope unmeasured rather than assumed unscaled. Every effective field is OMITTED, never zeroed, when the cvar is absent from the registry, so "not measured" stays distinguishable from "measured zero"; `expectedDrawnInstances` is documented as an expectation because `FClusterBuilder`'s exclusion is a per-instance random draw, exact only at 0 and 1. Ask #5 half-done: `foliage.add_type` gained an `enableDensityScaling` parameter (engine default false, preserved) and reports the flag read back off the asset; **`foliage.create_procedural` was deliberately left alone** — another agent was editing its per-type property block in the same wave and a second writer there would have clobbered it, so its per-type `density` still carries no cvar report. `foliage.remove`'s `instancesRemoved` was also left alone: a removal empties both the ledger and the component, so the density cull does not make that number mean something else. No cvar is ever written by a verb. Regression test `Tests/Environment/TestFoliageDensityScalabilityCVars.cpp` — `PinWright.foliage.paint.LedgerCountIsSeparatedFromWhatRenders` and `PinWright.landscape.create_grass_type.EffectiveDensityFollowsCVarScale`, both DIFFERENTIAL (same call, same fixture, only the cvar moved, effective figure must differ across the two runs) so a field echoed back from the request fails; each restores its cvar in an `ON_SCOPE_EXIT` at the priority the variable already had (`GetFlags() & ECVF_SetByMask`, because the engine's guard is `>=` not `>`) and tears down its `/Game/Foliage` and `/Game/Landscape` fixture assets. `check_test_ids.py`: CLEAN, 4748 ids, no dot-prefix collisions. NOT COMPILED and NOT RUN — the orchestrator owns the build and the suite.
