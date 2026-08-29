---
id: E-pv-graph-path-needs-optional-plugin-guard
title: "The Procedural Vegetation graph path needs a RUNTIME guard, not the __has_include pattern: PV ships with UE 5.8 on every host but is experimental and off by default, so a compile-time probe answers 'present' on a machine where the module is never loaded — and IntegrationGates keys on namespace and method prefix, so it cannot express a dependency carried by one parameter's value"
status: OPEN
severity: Medium
category: ergonomic
tags: [pcg, procedural-vegetation, optional-engine-plugin, integration-gates, has-include, plugin-disabled, build-cs, error-honesty, ue58]
blockedBy: [F-pcg-create-graph-class-parameter]
---

# Three existing patterns for optional engine plugins, and why the PV path fits none of them

`F-pcg-create-graph-class-parameter` asks for a `graphClass` parameter on `pcg.create_graph` so that
`UProceduralVegetationGraph` becomes reachable. Before that lands, this ticket settles how it must
behave on the overwhelmingly common host: one where the Procedural Vegetation Editor plugin is
present on disk and **disabled**.

`ProceduralVegetationEditor.uplugin` says so directly: `"IsExperimentalVersion": true`,
`"EnabledByDefault": false`, `"Installed": false`. Enabling it also pulls in four more plugins —
`Dataflow`, `GeometryScripting`, `PCG`, `DynamicWind` (its `"Plugins"` array). It is not a
reasonable thing to require.

## Pattern 1 — `__has_include` + feature macro (`WaterHandler.cpp`). Wrong here, and for a reason
worth writing down.

The canonical shape: `#if __has_include("WaterBodyActor.h")` -> `#define MCP_HAS_WATER 1/0`
(`Handlers/Water/WaterHandler.cpp:33-47`), `REGISTER_RPC_HANDLER` **unconditional** so the namespace
still appears in the wiki (`:131`), body branches on the macro (`:142`), `#else` sends a typed error
(`:181-184`: `WATER_PLUGIN_NOT_AVAILABLE`, *"Water plugin headers not compiled into PinWright"*), and
`Build.cs` soft-links with `TryAddConditionalModule(Target, EngineDir, "Water", "Water")`
(`PinWright.Build.cs:166`).

**`TryAddConditionalModule` probes the disk, not the project.** Its body
(`PinWright.Build.cs:272-300`) checks whether `Engine/Source/Runtime/<Name>`,
`Engine/Source/Editor/<Name>` or a plugins directory *exists*. It has no notion of whether the
consumer project has the plugin enabled. So a `__has_include`-derived macro answers **"the engine
shipped these headers"**, which for Procedural Vegetation is `true` on every stock UE 5.8 install —
including every install where the module is never loaded because the plugin is off. The guard would
compile the PV path in and then fail at runtime, which is strictly worse than not having it.

This is not a novel observation: `Source/PinWrightPCG/PinWrightPCG.Build.cs` says it in its own
comment — its `BuildException` guard *"does NOT cover the plugin-present-but-reference-disabled
case; that is the job of the `"Enabled": true` + `"Optional": true` ref in PinWright.uplugin (plus
the CI invariant check) and, at runtime, of IntegrationGates only loading this module when
IPluginManager reports PCG enabled."* PV is exactly the plugin-present-but-disabled case, by
default, on every host.

## Pattern 2 — quarantined sub-module + `IntegrationGates`. Right mechanism, wrong granularity.

`PinWrightPCG` is a `LoadingPhase: None` editor sub-module (`PinWright.uplugin:39-43`) that hard-links
PCG and is loaded at subsystem init only when `IPluginManager::Get().FindPlugin("PCG")->IsEnabled()`
(`Private/IntegrationGates.cpp:31` table row, loop at `:48-79`, invoked from
`PinWrightSubsystem.cpp:135`). When it is skipped, callers get a real answer rather than a typo
suggestion: `RpcDispatcher.cpp:690-698` returns `ERR_PLUGIN_DISABLED`
(`Handlers/ErrorCodes.h:1009`) with *"Method '%s' is unavailable because the '%s' engine plugin is
disabled in this project; enable it and restart the editor"*, and `WikiHandler.cpp:895` reports the
same for the namespace page.

This is the pattern the `pcg` surface already lives under, and it is the right *kind* of answer. But
its unit of gating is a **namespace slug** or a **method prefix** — the `FIntegrationEntry` fields
are `PluginName`, `ModuleName`, `ShortName`, `NamespaceSlug`, `MethodPrefix`
(`IntegrationGates.cpp:15-21`), and the lookups are `FindSkippedByMethod` / `FindSkippedByNamespace`
(`IntegrationGates.h:28`, `:34`). The PV dependency is not a namespace and not a method prefix:
`pcg.create_graph` works fine without PV, and needs PV **only when `graphClass` names a
`/Script/ProceduralVegetation.*` class**. No table row can say that.

## Pattern 3 — a second sub-module. Rejected.

A `PinWrightProceduralVegetation` module with its own `IntegrationGates` row would work
mechanically, but it would own no namespace and no method prefix — the verbs it would gate already
belong to `pcg` — so both lookup functions would return empty and the gate would produce no error
message. It also adds a `.uplugin` module, a `Build.cs`, a CI invariant entry and a load step to
support one optional parameter value.

## What to do instead

**Resolve the class by reflection and gate at runtime, in the handler.** This needs no `Build.cs`
change, no new module, and no compile-time dependency of any kind:

```cpp
UClass* GraphClass = ResolveUClass(GraphClassPath);        // Utils/ClassUtils, 3-tier
if (!GraphClass) { /* typed error, see below */ }
if (!GraphClass->IsChildOf(UPCGGraph::StaticClass())) { /* INVALID_ARGUMENT */ }
UPCGGraph* Graph = NewObject<UPCGGraph>(Package, GraphClass, FName(*AssetName), Flags);
```

A `/Script/<Module>.<Class>` path for a module that was never loaded simply does not resolve, which
is precisely the runtime signal the compile-time probe cannot produce. This is also the plugin's own
documented rule for engine `UCLASS` types it must not link against
(`agent-conventions.md`, "Lookalike-API traps": resolve by reflection, exported-base construction,
`IsA` + `static_cast`).

**Make the failure name the plugin.** A bare *"could not resolve class"* sends the caller hunting
for a typo in a path that is correct. Reuse the vocabulary `IntegrationGates` established — emit
`PLUGIN_DISABLED` (already registered, `ErrorCodes.h:1009`) when the unresolved path's `/Script/`
module maps to a known-optional engine plugin, with a message in the same shape as
`RpcDispatcher.cpp:692-695` plus the two facts a caller needs: the plugin is **experimental**, and
enabling it also enables `Dataflow`, `GeometryScripting`, `PCG` and `DynamicWind`.

**Then decide whether `IntegrationGates` should learn about parameter-value dependencies at all.**
Two defensible answers, and this ticket deliberately does not pick one:

- *Handler-local.* The gate's job is loading sub-modules and explaining absent **methods**; a
  dependency carried by one parameter's value is the handler's business. Cheapest, and keeps the
  gate's table meaning one thing.
- *Extend the gate.* Add a `FindSkippedByScriptModule(const FString& ScriptPath)` so the
  plugin-name-to-message mapping lives in one place and the next verb that resolves an engine class
  by path — there will be one — gets the same error for free. Costs a third lookup and a table
  column, and is the version that generalises to other engine plugins.

**Echo either way.** Whatever `pcg.create_graph` resolves, name it in the response (this is also
asked in `F-pcg-create-graph-class-parameter`), so a caller can tell "I got a PV graph" from "I got
the default because my path was ignored".

## Distinct from

- **`F-pcg-create-graph-class-parameter`** (OPEN, High) — the capability. Listed in `blockedBy`:
  there is no PV path to guard until it exists, and a picker should skip this until then. That
  ticket carries the *what*; this carries the *how it behaves when the plugin is off*, which is
  where a naive implementation does real damage (a hard link to `ProceduralVegetation` would break
  `PinWrightPCG`'s load, and with it the entire `pcg` namespace, on every host that has PV disabled
  — i.e. all of them by default).
- **`B-fab-hard-dll-imports-vs-optional-uplugin-refs`** (IN-REVIEW, Critical) — the prior art for why hard DLL imports of
  optional plugins are a problem here (the Fab `LoadLibrary` error 126 that `IntegrationGates.h:8-13`
  cites as its reason for existing). Same failure family, different plugin; this ticket is the
  application of that lesson to PV, one layer deeper than the gate currently reaches.

## Not RPC-verified

Source-read only; the editor was not running for this pass, and the Procedural Vegetation Editor
plugin was **not enabled**, so no PV class was resolved and no `PLUGIN_DISABLED` path was exercised.
The mechanism claims above are all from source. One thing an editor test would settle: whether
`FindObject<UClass>(nullptr, TEXT("/Script/ProceduralVegetation.ProceduralVegetationGraph"))` really
returns null when the plugin is disabled, or whether the module is loaded anyway by another enabled
plugin's dependency chain — PV is a *dependency of nothing* in a stock install, so null is expected,
but that is inference from the plugin graph rather than an observation.

severity rationale: impact=Medium — a design constraint on a dependent change rather than a defect in shipped behaviour; nothing is currently wrong because the path does not exist yet, and the cost of getting it wrong is either a confusing error (friction) or, in the hard-link version, a load failure that takes the whole `pcg` namespace down on every host with PV disabled — that second outcome is Critical-shaped, and it is the reason this is filed as its own gate rather than a bullet inside the feature ticket × reach=normal — the guarded path is one parameter on one verb, but the blast radius of the wrong implementation is the whole `pcg` sub-module, so the rare-path bump-down is declined -> Medium

## History
- `#1-runtime-not-compile-time-guard` `OPEN` reporter — Source-read only, editor not running, PV plugin not enabled; no PV class was resolved. Corrects the prescription this was proposed under: the canonical `WaterHandler.cpp` `__has_include` pattern (`:33-47`, `:131`, `:142`, `:181-184`; `PinWright.Build.cs:166`) is the WRONG guard for Procedural Vegetation, because `TryAddConditionalModule` (`PinWright.Build.cs:272-300`) probes whether the module directory exists on the build machine, not whether the consumer project enabled the plugin — and PV ships with every stock UE 5.8 (`"IsExperimentalVersion": true`, `"EnabledByDefault": false`, `"Installed": false` in `ProceduralVegetationEditor.uplugin`), so a compile-time probe answers "present" on precisely the hosts where the module is never loaded. `PinWrightPCG.Build.cs` states this limitation about itself in its own comment. The sub-module + `IntegrationGates` pattern (`PinWright.uplugin:39-43`; `IntegrationGates.cpp:15-21` table, `:31` PCG row, `:48-79` load; `PinWrightSubsystem.cpp:135`; `RpcDispatcher.cpp:690-698` and `WikiHandler.cpp:895` for `ERR_PLUGIN_DISABLED`, `ErrorCodes.h:1009`) is the right kind of answer but gates on `NamespaceSlug` / `MethodPrefix` (`IntegrationGates.h:28`, `:34`), and the PV dependency is carried by one parameter's VALUE — `pcg.create_graph` works fine without PV and needs it only when `graphClass` names `/Script/ProceduralVegetation.*`. A second sub-module is rejected: it would own no namespace and no method prefix, so both lookups would return empty and it would produce no message. Recommendation is reflection resolution in the handler (no link dependency at all, per `agent-conventions.md`'s rule for unexported engine `UCLASS` types) plus a `PLUGIN_DISABLED` error naming the plugin, its experimental status, and the four plugins it drags in; whether `IntegrationGates` should grow a `FindSkippedByScriptModule` lookup is argued both ways and deliberately left open. `blockedBy: F-pcg-create-graph-class-parameter` — there is nothing to guard until that lands. Dedup: searched the board for `__has_include`, `TryAddConditionalModule`, `IntegrationGates`, `PLUGIN_DISABLED`, `optional plugin`, `experimental` and `ProceduralVegetation`. Zero hits for Procedural Vegetation anywhere. `B-fab-hard-dll-imports-vs-optional-uplugin-refs` is the closest — same failure family (hard DLL imports of optional plugins) and cross-linked — but is about the existing sub-module refs, not about a per-parameter-value dependency, and predates any PV path.
