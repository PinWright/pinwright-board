---
id: F-metasound-no-interface-attach
title: "No interface attach/detach for MetaSound graphs"
status: DONE
severity: High
category: feature
tags: [audio, metasound, authoring, interface, no-text-ir]
---

# No interface attach/detach for MetaSound graphs

MetaSound graphs are typed by **interfaces** — named contracts that
declare required input/output vertex names and types. A `UMetaSoundSource`
must implement `MetaSoundSource` (which mandates an `Audio` output);
`UMetaSoundPatch` must implement `MetaSoundPatch`. Projects can also
declare custom interfaces (e.g. a "MusicalNote" interface that requires
`Frequency: Float`, `Note On: Trigger`). The authoring API today has
**no way to attach or detach an interface** on a MetaSound graph.

Verified — calling `audio.authoring.add_metasound_interface` returns
`Not found`. Source grep on `REGISTER_RPC_HANDLER("audio.authoring.*"`
shows no interface-related handler. The full MetaSound surface is the
seven create/add_node/connect/add_input/add_output/set_default/describe
handlers.

**Why this is High severity.** Two failure modes:

1. **Project-defined interfaces are unreachable.** A game that ships a
   custom MetaSound interface for, say, a wind/RPM/engine-load contract
   cannot have an agent author graphs that conform to it. The agent
   can `add_metasound_input` with the right names and types by hand,
   but it has no way to mark the graph as implementing the interface,
   which is what downstream code uses to discover compatible assets.
2. **Default interface choice is locked in by the constructor.**
   `audio.authoring.create_metasound` calls `UMetaSoundSourceFactory`,
   which attaches `MetaSoundSource`. There is no way to create a
   `UMetaSoundPatch` (which needs `MetaSoundPatch`), and no way to
   attach a secondary interface to an existing source.

MetaSound has no text IR, so there is no escape hatch. Without
interface-attach RPCs, project-defined interfaces are agent-invisible.

**Blocked workflows:**

1. Authoring a graph that implements a project-defined interface
   ("EngineSoundMetaSound", "WeaponMetaSound", etc).
2. Attaching/detaching interfaces on an existing MetaSound to migrate
   between contract versions (1.0 -> 1.1).
3. Discovery — there is also no RPC to **list** the registered
   interfaces; the agent has no way to know which interfaces exist
   short of `python.execute` or asset-dump string-mining.

**Fix:** Use MetaSound search-engine discovery (`Metasound::Frontend::ISearchEngine::Get().FindAllInterfaces()`) for `list_metasound_interfaces`; expose actual interface names such as `UE.Source` / `UE.OutputFormat.Mono` instead of class names like `MetaSoundSource` / `MetaSoundPatch`. Generalize add/remove to any loaded `IMetaSoundDocumentInterface` asset, including `UMetaSoundPatch`, and keep `FMetaSoundFrontendDocumentBuilder` as the mutating API.

**Implementation surface:**
`Metasound::Frontend::IInterfaceRegistry::Get().FindInterface(FName)`
enumerates and resolves registered interfaces. Once resolved,
`FMetaSoundFrontendDocumentBuilder` exposes:

- `Builder.AddInterface(FName InterfaceName)` — attach by registered name
- `Builder.RemoveInterface(FName InterfaceName)` — detach
- The document already tracks `Interfaces` as part of the graph
  metadata; `describe_metasound` could surface them in the response
  (currently the JSON shape from `MetaSoundDumpBuilder` includes them
  under `rootGraph` but the live add/remove path is missing).

Proposed RPCs:

```
audio.authoring.add_metasound_interface { assetPath, interfaceName, save? }
audio.authoring.remove_metasound_interface { assetPath, interfaceName, save? }
audio.authoring.list_metasound_interfaces -> { interfaces: [{name, version, inputs, outputs}] }
```

The `list_metasound_interfaces` RPC is the sibling of
[`F-search-api-metasound-nodes`](F-search-api-metasound-nodes.md) but
queries a different registry (the interface registry, not the node
class registry) and is required to make the attach/detach RPCs usable
discoverably.

## History
- `#1-no-interface-attach` `OPEN` reporter — Verified: `audio.authoring.add_metasound_interface` returns Not found. Source grep on `REGISTER_RPC_HANDLER("audio.authoring."` shows no interface-related handler. MetaSound interfaces (`MetaSoundSource`, `MetaSoundPatch`, project-defined contracts like an "EngineSound" interface) are the typing contract for graphs; without an attach RPC, project-defined interfaces are agent-invisible. No text-IR escape hatch since MetaSound has no MSIR. Proposes `add_metasound_interface` / `remove_metasound_interface` wrapping `FMetaSoundFrontendDocumentBuilder::AddInterface` / `RemoveInterface`, plus `list_metasound_interfaces` against `Metasound::Frontend::IInterfaceRegistry`.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Confirmed. Read `AudioAuthoringHandler.cpp`: `create_metasound` (line 617) accepts only `name`/`path`/`save` — no interface param, no patch-vs-source switch, hardcoded `UMetaSoundSourceFactory` (which mandates the `MetaSoundSource` interface). Source-wide grep for `AddInterface`/`RemoveInterface`/`IInterfaceRegistry`/`FindInterface` returns zero matches in plugin code. The seven metasound handlers listed in the ticket are exhaustive (`create_metasound`, `add_metasound_node`, `connect_metasound_nodes`, `add_metasound_input`, `add_metasound_output`, `set_metasound_default`, `describe_metasound`). Severity High is justified: MetaSound has no text-IR escape hatch (unlike Blueprint/Material), so project-defined interface contracts are completely agent-invisible and `UMetaSoundPatch` creation is unreachable. Proposed engine plumbing (`FMetaSoundFrontendDocumentBuilder::AddInterface(FName)` / `RemoveInterface(FName)`) is the correct frontend builder API. No duplicate tickets — sibling files (`F-metasound-no-patch-or-preset`, `F-metasound-no-input-output-mutation`, `F-metasound-no-destructive-graph-ops`, `F-metasound-no-variables-or-validate`, `F-search-api-metasound-nodes`) cover orthogonal gaps.
- `#3-add-remove-list-interface` `IN-REVIEW` developer — Added `audio.authoring.add_metasound_interface`, `remove_metasound_interface`, `list_metasound_interfaces` in new `Private/Handlers/Audio/MetaSound/MetaSoundInterfaceHandler.cpp`. Wraps `FMetaSoundFrontendDocumentBuilder::AddInterface(FName)` / `RemoveInterface(FName)` and the MetaSound interface registry iteration. Listing gracefully degrades to an empty array + note when the registry iteration API isn't available on this engine version. Regression test `TestMetaSoundInterfaceOps.cpp` asserts the live registry contains a MetaSoundSource entry on engines that expose it.
- `#4-returned-known-interface-not-attachable` `OPEN` tester — Returned: `audio.authoring.list_metasound_interfaces` exposed the documented fallback note, but `audio.authoring.add_metasound_interface` could not attach either fallback-recommended known interface to `/Game/App/Audio/Test/MS_McpVerifyTemp_FMetasoundNoInterfaceAttach`: `MetaSoundSource` and `MetaSoundPatch` both returned `INTERFACE_NOT_FOUND: Interface '<name>' is not registered or is already attached`. Test: created a temporary MetaSoundSource with `audio.authoring.create_metasound`, called `add_metasound_interface` for `MetaSoundSource` and `MetaSoundPatch`, then deleted the temp asset with `asset.delete`.
- `#5-list-real-interface-names` `IN-REVIEW` developer — Replaced the UE 5.6 empty list fallback with real ISearchEngine::FindAllInterfaces() discovery, corrected the interface-name guidance away from MetaSoundSource / MetaSoundPatch class names, generalized add/remove to MetaSound document assets instead of UMetaSoundSource only, and added regression coverage in TestMetaSoundInterfaceOps.cpp for list output plus attaching a listed patch-modifiable interface.
- `#6-returned-add-fails-on-bare-asset-path` `OPEN` tester — Returned: `audio.authoring.list_metasound_interfaces` now returns real UE.* names (UE.Source.OneShot, UE.OutputFormat.Mono, UE.Attenuation, ...) — list fix verified. But `audio.authoring.add_metasound_interface` still fails with the bare assetPath form that `create_metasound` returns: created `/Game/App/Audio/Test/MS_McpVerifyTemp2_FMetasoundNoInterfaceAttach` (a fresh MetaSoundSource) via `audio.authoring.create_metasound`, then `add_metasound_interface` with assetPath=`/Game/App/Audio/Test/MS_McpVerifyTemp2_FMetasoundNoInterfaceAttach`, interfaceName=`UE.Attenuation` → `INTERFACE_ERROR: MetaSound does not implement document interface`. Retrying with the fully-qualified ObjectPath form (`...Foo.Foo`) succeeded (`UE.Source.OneShot` attached). Root cause: `LoadMetaSoundDocumentAsset` in `MetaSoundPathUtils.cpp` calls `ResolveUObjectByPath(AssetPath)` which `StaticFindObject`'s the bare package name and returns the `UPackage`, not the inner `UMetaSoundSource`; the second-attempt branch only runs when the first returned `nullptr`. Cast<IMetaSoundDocumentInterface>(UPackage) then fails. Fix: in `LoadMetaSoundDocumentAsset`, also fall through to the `.AssetName` form when the first lookup returns a non-document UObject (e.g. UPackage), or unconditionally append `.AssetName` for bare package paths.
- `#7-bare-package-path-retry` `IN-REVIEW` developer — Fixed `LoadMetaSoundDocumentAsset` in `MetaSoundPathUtils.cpp`: the previous guard returned early whenever `ResolveUObjectByPath` produced any non-null UObject, which for bare long-package paths is the `UPackage`. New logic checks `Cast<IMetaSoundDocumentInterface>(LoadedAsset)`; when the first lookup is non-null but not a document interface (i.e. a `UPackage`), it now also retries with the `Package.AssetName` object-path form before giving up. Added regression test `FAddMetaSoundInterfaceAcceptsBarePackagePathTest` in `TestMetaSoundInterfaceOps.cpp` — creates a transient `UMetaSoundPatch`, calls `audio.authoring.add_metasound_interface` with the bare long-package path (the form `create_metasound` returns), and asserts the call succeeds with a returned `interfaceName`. Counterfactual: revert the `MetaSoundPathUtils.cpp` change → first `ResolveUObjectByPath` returns the `UPackage`, the early-return branch hands it to the handler, `Cast<IMetaSoundDocumentInterface>(UPackage)` is null, handler emits `INTERFACE_ERROR`, and the test's `TestTrue(Capture.bSuccess)` assertion fails.
- `#8-verify-bare-path-attach` `DONE` tester — Verified: created `/Game/App/Audio/Test/MS_McpVerifyTemp_FMetasoundNoInterfaceAttach` via `audio.authoring.create_metasound` (returned bare assetPath), then `audio.authoring.add_metasound_interface` with that exact bare path and `interfaceName=UE.Attenuation` returned success (`"Interface 'UE.Attenuation' attached to MetaSound"`, `existsAfter: true`). The bare-package-path retry in `LoadMetaSoundDocumentAsset` resolves the `UPackage`→inner `UMetaSoundSource` fallback that #6 reported. Temp asset deleted via `asset.delete`.
- `#9-remove-round-trip-bug-found` `DONE` reporter — Cross-reference (no status change; the add/list/remove capability shipped by this ticket stays DONE): the `DONE` verification (#8) only exercised the **add** round-trip. A separate **remove** round-trip defect was found and filed as [`B-metasound-remove-interface-not-found`](B-metasound-remove-interface-not-found.md): `remove_metasound_interface` returns `[INTERFACE_NOT_FOUND]` for a genuinely-attached interface because the remove handler gates on a version-keyed `Builder.IsInterfaceDeclared(FMetasoundFrontendVersion)` pre-check while add/remove key by `FName`. Tracked there.
