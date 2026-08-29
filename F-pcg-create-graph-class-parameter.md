---
id: F-pcg-create-graph-class-parameter
title: "pcg.create_graph hardcodes NewObject<UPCGGraph>, so UE 5.8's Procedural Vegetation Editor — a UPCGGraph subclass whose every node is a UPCGSettings the existing pcg.add_node already resolves by path — is unreachable through PinWright for want of one optional graphClass parameter"
status: IN-REVIEW
severity: High
category: feature
tags: [pcg, create-graph, procedural-vegetation, vegetation, foliage, tree-authoring, ue58, graph-class, missing-parameter, engine-plugin]
---

# One hardcoded `NewObject` is the whole distance between PinWright and native tree authoring

`pcg.create_graph` constructs exactly one class and cannot be told otherwise:

```cpp
UPCGGraph* Graph = NewObject<UPCGGraph>(
    Package,
    FName(*AssetName),
    RF_Public | RF_Standalone | RF_Transactional);
```

`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphCreate.cpp:73-76`. The verb's parameter list is
`name` + `savePath` and nothing else (`:23-24`). There is no way for a caller to ask for a
`UPCGGraph` **subclass**.

UE 5.8 ships one that matters for vegetation work.

## The chain, and how much of it already works

**1. The graph type is a `UPCGGraph` subclass.**

```cpp
UCLASS(MinimalAPI)
class UProceduralVegetationGraph : public UPCGGraph
```

`C:/UE_5.8/Engine/Plugins/Experimental/ProceduralVegetationEditor/Source/ProceduralVegetation/Public/ProceduralVegetation.h:8-9`.
Its constructor is inline in that public header and sets three editor flags
(`bIsStandaloneGraph = false`, `bExposeGenerationInAssetExplorer = false`, and both hidden-flag
input/output node calls, `:14-28`) — nothing that needs a factory to run. **Blocked today**, and
this is the only blocked link in the chain.

**2. Every Procedural Vegetation node is a `UPCGSettings`.**

```cpp
UCLASS(BlueprintType, Abstract, HideCategories=(Debug, AssetInfo), ClassGroup = (Procedural))
class UPVBaseSettings : public UPCGSettings, public IPVRenderSettings
```

`.../ProceduralVegetation/Public/Nodes/PVBaseSettings.h:11-12`. Every PV node derives from it.

**3. `pcg.add_node` already resolves any such class by path, with no allowlist.**

`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphAuthoring.cpp`:

```
:40  RPC_PARAM_REQ("nodeClass", "string", "UPCGSettings subclass path, e.g. /Script/PCG.PCGCreatePointsSettings.")
:62  UClass* SettingsClass = FindObject<UClass>(nullptr, *NodeClassPath);
:65  SettingsClass = LoadClass<UPCGSettings>(nullptr, *NodeClassPath);
:67  if (!SettingsClass || !SettingsClass->IsChildOf(UPCGSettings::StaticClass()))
:75  UPCGNode* Node = Graph->AddNodeOfType(TSubclassOf<UPCGSettings>(SettingsClass), DefaultSettings);
```

The only gate is `IsChildOf(UPCGSettings)`. There is no module allowlist, no path prefix check, no
registry of known node types. **Header visibility is irrelevant to this path**: UHT registers a
`UCLASS` under `/Script/<Module>.<Class>` whether its header sits in `Public/` or `Private/`, so
private PV node classes — `UPVGrowerAuxinSettings`
(`.../Private/Nodes/GrowerSettings/PVGrowerAuxinSettings.h:9-10`),
`UPVGrowerBifurcationSettings` (`.../PVGrowerBifurcationSettings.h:9-10`), and the ~30 siblings in
`Private/Nodes/` — resolve by `/Script/ProceduralVegetation.<ClassName>` exactly like a stock PCG
node. **Already works, unblocked, needs no change.**

**4. The graph's terminal node is public.** `UPVExportSettings : public UPVBaseSettings`
(`.../Public/Nodes/PVExportSettings.h:9-10`), so the export step is reachable, and `pcg.generate`
already drives generation.

So: link 1 is one line, links 2–4 need nothing. That asymmetry is the whole argument for this
ticket — this is not "add Procedural Vegetation support", it is "stop hardcoding the graph class",
and the rest of the surface has already been built.

## Correction to an earlier draft of this finding

An earlier write-up of this research named `/Script/ProceduralVegetation.PVGrowerPhyllotaxySettings`
as the private-node example. **No such class exists.** Phyllotaxy in PV is a *struct*,
`FPVHormonePhyllotaxySettings`, embedded in distribution parameters
(`.../Private/DataTypes/PVDistributionParams.h:388-405`) — it is a settings block on a node, not a
node. The private-node point stands on the real classes cited above; the example was wrong and is
corrected here so nobody spends an editor session resolving a class name that was never there.

## The ask

One **optional** parameter on `pcg.create_graph`:

    graphClass (string, optional) — UPCGGraph subclass path. Defaults to /Script/PCG.PCGGraph.

Resolved with the same three-tier discipline the plugin already mandates for engine classes
(`Utils/ClassUtils::ResolveUClass`; and `agent-conventions.md` on engine `UCLASS` types without an
export macro — resolve by reflection, never by `StaticClass()` linkage), gated by
`IsChildOf(UPCGGraph::StaticClass())` exactly as `pcg.add_node:67` gates its node class, and
constructed via the `NewObject(Package, ResolvedClass, Name, Flags)` overload. **Deliberately by
reflection and not `NewObject<UProceduralVegetationGraph>`**: the PV plugin is off by default (see
`E-pv-graph-path-needs-optional-plugin-guard`), so a compile-time reference would put a hard link
dependency on an optional experimental module into `PinWrightPCG`.

Echo the resolved class in the response next to `graphPath`, so a caller can tell a defaulted
`UPCGGraph` from the subclass they asked for. Not echoing it would reproduce, on a brand-new
parameter, the response-honesty defect this board is full of.

## Not RPC-verified

Source-read only. The editor was not running for this pass; no `pcg.create_graph` call was made and
no PV plugin was enabled. Two questions an editor test would settle, and this analysis cannot:

1. **Is `NewObject<UProceduralVegetationGraph>` sufficient without the PV factory?** The class is
   `MinimalAPI` and its constructor is inline, so construction should link and run; but the PV
   editor has its own asset factory, and whether it seeds nodes, a preset, or asset-registry state
   that a bare `NewObject` skips is unknown. If it does, this ticket's ask is still correct but the
   resulting graph may need `pcg.add_node` calls the PV editor would have made for you.
2. **Do PV nodes execute outside the PV editor UI?** Every PV node is a `UPCGSettings` with an
   `FPVBaseElement : IPCGElement`, so `pcg.generate` *should* run them on the ordinary PCG
   scheduler. Whether any of them depends on PV-editor-only state (a preview world, a growth-data
   loader path, the `SPVExportSelectionDialog` flow at
   `.../ProceduralVegetationEditor/Private/Widgets/SPVExportSelectionDialog.h`) is unverified. If
   they do, the useful subset of PV through PinWright is smaller than the class hierarchy suggests
   — which would change this ticket's *value*, not its correctness.

Neither question changes the fix; both change how much the fix buys. Answer them in the same editor
session that verifies the fix.

## Distinct from

- **`F-pcg-core-graph`** (DONE) — shipped the namespace this parameter would extend. That ticket
  delivered `pcg.create_graph` in its current, single-class form; this is the follow-on, not a
  reopening.
- **`F-pcg-set-node-property`** (OPEN, Low) — the *other* half of the same capability. With
  `graphClass` you can create a PV graph and add PV nodes; without per-node property writes you
  cannot configure any of them, and a PV node with default parameters is a species you did not ask
  for. That ticket is recommended for re-severity to High partly because of this one; the two are
  independently valuable and independently testable, which is why they stay separate.
- **`E-pv-graph-path-needs-optional-plugin-guard`** (OPEN, Medium) — how the PV path must behave on
  the overwhelmingly common host where the PV plugin is disabled. That is a hard prerequisite for
  landing this safely, so it is filed as a blocker rather than folded in here.
- **`F-pcg-authoring-parity`** (IN-REVIEW), **`F-pcg-filters-and-subgraphs`** (DONE),
  **`F-pcg-generate-readback`** (IN-REVIEW) — graph *content* asks. This one is about the graph
  object's type, which is upstream of all of them.
- **`E-pcg-add-node-echo-pin-labels`** (OPEN, Low) — response-shape gap on `create_graph` and the
  node-create verbs. Touches the same two files. A fixer in `PCGGraphCreate.cpp` for either should
  read the other, since both add fields to the same response.
- **No umbrella "support Procedural Vegetation" ticket is proposed.** Support decomposes into this
  ticket, `F-pcg-set-node-property`, and `E-pv-graph-path-needs-optional-plugin-guard` — each with
  a concrete code site and an independent test. An umbrella would be a wish with no repro on a
  board whose convention is repro-backed items.

## Note for whoever opens `PCGGraphCreate.cpp`

The whole file sits inside `#if defined(__has_include) && __has_include("PCGGraph.h")` (`:7`,
`#endif` at `:94`), which puts the `REGISTER_RPC_HANDLER` at `:20` inside the guard — the opposite
of `agent-conventions.md`'s "registration is ALWAYS unconditional". **This is not a defect and does
not need fixing.** The `pcg` namespace is gated a level higher: `PinWrightPCG` is a
`LoadingPhase: None` sub-module (`PinWright.uplugin:39-43`) loaded only when `IPluginManager`
reports PCG enabled (`Private/IntegrationGates.cpp:31`), and `IntegrationGates::FindSkippedByNamespace`
supplies the wiki entry and the `PLUGIN_DISABLED` error in the module's absence. Checked so the next
reader does not re-derive it.

severity rationale: impact=High within the "hard blocker with no workaround" band — a shipped UE 5.8 authoring subsystem is unreachable through the plugin's own surface, and the block is one hardcoded template argument rather than any missing capability; taking the High end rather than Medium because what is blocked is an entire engine subsystem, not one verb, and because two other tickets depend on it × reach=normal — `pcg.create_graph` is the gateway verb of its namespace (no other `pcg` verb applies until a graph exists), so the rare-edge-path bump-down is declined; a reviewer who scores `python.execute` as a general workaround (it can `NewObject` a PV graph directly) lands on Medium, and that is the specific argument being rejected here, because "the plugin had a gap so we left the plugin" is the outcome this project treats as the defect -> High

## History
- `#1-hardcoded-graph-class` `OPEN` reporter — Source-read only, editor not running; no `pcg.create_graph` call was made and the PV plugin was not enabled. `PCGGraphCreate.cpp:73` hardcodes `NewObject<UPCGGraph>` and the verb declares only `name`/`savePath` (`:23-24`), so no `UPCGGraph` subclass is reachable. UE 5.8's `UProceduralVegetationGraph` is one (`ProceduralVegetation.h:8-9`, `UCLASS(MinimalAPI)`, inline constructor), every PV node is a `UPCGSettings` via `UPVBaseSettings` (`PVBaseSettings.h:11-12`), and `pcg.add_node` already resolves any `UPCGSettings` subclass by `/Script/...` path with `IsChildOf` as its only gate and no module allowlist (`PCGGraphAuthoring.cpp:62`, `:65`, `:67`, `:75`) — including classes in private headers, since UHT registers by name not by header visibility. Ask is one optional `graphClass` parameter resolved by reflection (not `NewObject<T>`, to avoid a link dependency on an off-by-default experimental module) and echoed in the response. Corrects an earlier draft of this research: `UPVGrowerPhyllotaxySettings` does not exist — phyllotaxy is the struct `FPVHormonePhyllotaxySettings` (`PVDistributionParams.h:388`); real private node examples are `UPVGrowerAuxinSettings` and `UPVGrowerBifurcationSettings`. Dedup: checked every `F-pcg-*`, `B-pcg-*` and `E-pcg-*` file on the board — `F-pcg-core-graph` (DONE) shipped `create_graph` in its single-class form, `F-pcg-authoring-parity` (IN-REVIEW), `F-pcg-filters-and-subgraphs` (DONE) and `F-pcg-generate-readback` (IN-REVIEW) are graph-content asks, `E-pcg-add-node-echo-pin-labels` (OPEN) is a response-shape gap in the same two files, and `B-pcg-generate-deadlocks-game-thread` / `B-pcg-inspect-edge-direction-reversed` / `B-pcg-generated-graph-output-empty-after-generate` are unrelated defects. No ticket mentions `UProceduralVegetationGraph`, `graphClass`, Procedural Vegetation, or a graph-class parameter. Two questions left for an editor test and named in the body: whether `NewObject<UProceduralVegetationGraph>` suffices without the PV factory, and whether PV nodes execute outside the PV editor UI.
- `#2-graphclass-parameter-landed` `IN-REVIEW` fixer — Implemented as filed; premise verified in source first. `PCGGraphCreate.cpp` gains `RPC_PARAM_DEF("graphClass", "string", …, "/Script/PCG.PCGGraph")`, resolves it through `Utils/ClassUtils::ResolveUClass` (the plugin's shared full-path-or-short-name resolver, same helper `asset.create_blueprint`'s `parentClass` uses; deliberately not `NewObject<T>`, so no link dependency on an optional module), gates on `IsChildOf(UPCGGraph)`, and passes the class to the `NewObject(Package, Class, Name, Flags)` overload. Resolution happens **before** `CreatePackage`, so a rejected class leaves no package, no asset and no registry entry. Response now echoes `graphClass` read off `Graph->GetClass()->GetPathName()` — off the created object, not off the request. **One addition beyond the ask, and it is not optional:** the class is also rejected when `CLASS_Abstract` is set, because `NewObject`'s own abstract guard is a `checkf` (`UObjectGlobals.cpp` `StaticConstructObject_Internal`, `IsDisallowedAbstractClass`), i.e. an editor crash rather than a null return — an abstract `UPCGGraph` subclass reaching `NewObject` would have taken the host down. **Class-shape question the ticket left open, answered: a class parameter is right and a template/parent-graph parameter would be wrong.** PCG has no graph inheritance — `bIsTemplate` (`PCGGraph.h:243`, on `UPCGGraphInterface`) is a bool marking an asset as a picker template, not a parent to derive from; `UPCGGraphInstance` is a *sibling* of `UPCGGraph` under `UPCGGraphInterface`, not a subclass, so it is unreachable through this verb by construction and correctly so; and duplicating an existing graph is already `asset.duplicate`. Confirmed by engine sweep that `UProceduralVegetationGraph` is the **only** `: public UPCGGraph` in the whole UE 5.8 tree. **Overlap with the sibling ticket `E-pv-graph-path-needs-optional-plugin-guard`, landed concurrently in this same wave:** that work added `PinWrightPCG::SendPluginDisabledForScriptPath` to `PCGHandlerHelpers.h` (plus a `Projects` module dep in `PinWrightPCG.Build.cs`) and wired it into `pcg.add_node`. `graphClass` reuses it on the identical failure path rather than duplicating the probe, so a caller naming `/Script/ProceduralVegetation.ProceduralVegetationGraph` on the overwhelmingly common host where that experimental plugin is disabled gets `PLUGIN_DISABLED` naming the plugin (and, from the descriptor, that it is experimental and pulls in Dataflow/GeometryScripting/PCG/DynamicWind) instead of a misleading `CLASS_NOT_FOUND`. This creates a build-order dependency on that ticket's helper. No `#if` / `__has_include` guard of my own was invented — the file's existing whole-file `__has_include("PCGGraph.h")` guard is untouched, and the runtime plugin question is answered by that shared runtime probe, per the module's established pattern. Regression test `PinWright.pcg.create_graph.GraphClassSelectsSubclass` (`Source/PinWrightPCG/Private/Tests/PCG/TestPCGGraphHandlers.cpp`), four blocks, the first three of which fail against the pre-change handler: (1) `ParamSpecTestHelpers::IsParamAccepted` proves the dispatcher accepts the key — necessary because `InvokeHandlerWithCapture` never runs `ValidateHandlerParams`, so a body-level read would otherwise "pass" for a parameter no caller could supply; (2) omitting `graphClass` still builds a stock `UPCGGraph` *and* echoes it; (3) `graphClass: /Script/Engine.Actor` is refused with `CLASS_NOT_FOUND` and leaves no asset — the pre-change handler ignored the key and created one; (4) the ticket's actual claim, that a non-default class yields an asset **of that class**, asserted against any concrete `UPCGGraph` subclass discovered by reflection, emitting `PINWRIGHT_ASSERTIONS_SKIPPED` (`reason=no-upcggraph-subclass-loaded`) on a host where the PV plugin is off so the unmeasured assertion is visible rather than silently green. `Docs/wiki-src/pcg.md` gains one `## Conventions` bullet (above the first `###`, so the wiki-src rendering lint is satisfied). Not compiled and not RPC-verified — this wave does not build. **Both questions the ticket left for an editor session remain open and are what review should answer:** whether a bare `NewObject` of `UProceduralVegetationGraph` suffices without the PV asset factory, and whether PV nodes execute under `pcg.generate` outside the PV editor UI. Neither affects correctness of this change; both bound its value. **Kept separate from `F-pcg-set-node-property` deliberately — no shared root cause.** That ticket's blocker is in a different file and of a different kind: `pcg.inspect` emits `settingsClass` (a *class* path, `PCGGraphInspect.cpp` `NodeToJson`) and never the settings sub-object's *object* path, which is what makes its documented `property.set` workaround unaddressable; its fix is a new verb plus an inspect field. Verified in source this pass — the `#2` re-severity rationale on that ticket is accurate. The two share a use case (assemble a PV graph, then configure its nodes), not a mechanism, and no single change serves both.
- `#3-cannot-be-settled-on-this-host` `IN-REVIEW` tester — **No verdict. Status deliberately unchanged: this is neither a verification nor a return.** The implementation is correct by construction, but the ticket's central assertion — that a non-default `graphClass` yields an asset **of that class** — is a permanently skipped test on this host, so nothing available here can settle it in either direction. Marking it `DONE` would claim an observation nobody made; returning it to `OPEN` would blame the developer for a plugin that is off. Recording the state instead, so the next tester does not spend the pass rediscovering it.

  **What was checked and holds, on inspection at HEAD.** The `graphClass` resolution path is present and correct: `Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphCreate.cpp:47-48` defaults `GraphClass` to `UPCGGraph::StaticClass()` and reads the trimmed parameter, `:51` resolves it through `ResolveUClass` (reflection, no `NewObject<T>`, so no link dependency on an optional module), `:61-63` gates on `!GraphClass || !IsChildOf(UPCGGraph::StaticClass()) || HasAnyClassFlags(CLASS_Abstract)` and answers `CLASS_NOT_FOUND`, and the whole block sits **before** `CreatePackage`, so a rejected class leaves no package and no registry entry as `#2` claimed. The `CLASS_Abstract` guard is real and its stated reason is right — `NewObject`'s own abstract check is a `checkf`, so an abstract subclass reaching it would take the editor down rather than return null. The `PLUGIN_DISABLED` routing is wired at `:52-57`, calling the sibling ticket's shared `PinWrightPCG::SendPluginDisabledForScriptPath` on the identical failure path rather than duplicating the probe; that helper is separately **verified live** — see `E-pv-graph-path-needs-optional-plugin-guard` `#3`. The echo is at `:132`, `Graph->GetClass()->GetPathName()`, read off the created object exactly as `#2` describes.

  **Why it cannot be verified here.** `system.inspect.search_classes` for `UPCGGraph` subclasses returns **`totalMatches: 1`**, and that one match is **`PCGGraph` itself** — **no concrete `UPCGGraph` subclass exists in any loaded module on this host**. A fixture cannot be authored around that, because `UPCGGraph` is **not `Blueprintable`**: `C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Public/PCGGraph.h:331` declares `UCLASS(MinimalAPI, BlueprintType, ClassGroup = (Procedural), hidecategories=(Object))` — `BlueprintType` makes it usable as a variable type, and without `Blueprintable` no Blueprint can derive from it. **The plugin's own test acknowledges exactly this and skips honestly.** In `Source/PinWrightPCG/Private/Tests/PCG/TestPCGGraphHandlers.cpp`, `PinWright.pcg.create_graph.GraphClassSelectsSubclass` (declared `:378-379`, `RunTest` `:386`) sweeps `TObjectIterator<UClass>` for a non-abstract `UPCGGraph` subclass at `:461-473`, and at `:475-482` — when it finds none — calls `PinWrightTestSkip::SkipAssertions(*this, TEXT("no-upcggraph-subclass-loaded"), …)` at `:477`, whose message reads: *"No concrete UPCGGraph subclass is loaded on this host (UE 5.8 ships only UProceduralVegetationGraph, whose plugin is disabled by default), so the non-default-class assertion could not run."* Verified present, and the reason string is `no-upcggraph-subclass-loaded` verbatim. Blocks 1-3 of that test (dispatcher accepts the key; omission still builds and echoes a stock `UPCGGraph`; `/Script/Engine.Actor` refused leaving no asset) are not skipped and do exercise real behaviour — the skip is scoped to block 4 alone. **A skipped test here is the expected state, not a coverage failure:** the parameter's entire value is conditional on an optional experimental plugin, and `#2` chose to make the unmeasured assertion visible rather than let it pass green, which is the right call.

  **What would settle it, concretely.** Enable the **Procedural Vegetation Editor** plugin — it ships with stock UE 5.8, `"IsExperimentalVersion": true`, `"EnabledByDefault": false` — and re-run with `graphClass` naming `UProceduralVegetationGraph`, the only concrete subclass this ticket was ever written for and, per `#2`'s engine sweep, the only `: public UPCGGraph` in the whole 5.8 tree. **Re-run both tickets together, not this one alone:** enabling that plugin also converts `E-pv-graph-path-needs-optional-plugin-guard`'s `PLUGIN_DISABLED` path from the branch that is currently tested to the branch that currently is not, so its `#3` verdict is scoped to the disabled host and would need re-establishing on the enabled one. That session should also close `#1`'s and `#2`'s two standing questions, which this entry does not touch: whether a bare `NewObject` of `UProceduralVegetationGraph` suffices without the PV asset factory, and whether PV nodes execute under `pcg.generate` outside the PV editor UI.

  **A doc defect found while checking, filed separately as `E-pcg-graph-class-echo-not-provenance`.** `Docs/wiki-src/pcg.md:12` says the resolved class "is echoed back as `graphClass` **so a default is distinguishable from a requested subclass**". It is not: omitting the parameter and passing `graphClass: "PCGGraph"` produce **byte-identical responses**. That is a necessary consequence of `#2`'s own decision — correct, and not being questioned — to read the echo off `Graph->GetClass()` (`:132`) rather than off the request; an echo of the created object's class distinguishes **classes**, not **provenance**, and the two coincide only when the requested class differs from the default. The implementation is right and the doc sentence overclaims what it can deliver. Recorded honestly: that ticket file did not exist on the board when this entry was written.
