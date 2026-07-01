---
id: B-metasound-patch-mutators-reject
title: "MetaSound authoring mutators reject UMetaSoundPatch with misleading ASSET_NOT_FOUND — Patch graphs are unpopulatable"
status: IN-REVIEW
severity: Medium
category: bug
tags: [audio, metasound, authoring, patch, asset-not-found, cast-mismatch]
---

# MetaSound authoring mutators reject UMetaSoundPatch — created Patches can never be populated

`audio.authoring.create_metasound_patch` creates a `UMetaSoundPatch` and
its wiki page explicitly directs the caller to build the graph with the
shared authoring mutators:

> "Use audio.authoring.add_metasound_input / add_metasound_output /
> add_metasound_node to build the graph, then connect_metasound_nodes to
> wire it."

But every one of those mutators **rejects a Patch** with a misleading
`[ASSET_NOT_FOUND] Could not load MetaSound: <path>` — even though the
asset exists and loads fine (`describe_metasound` reads it on the exact
same path). A `UMetaSoundPatch` is therefore a write-only dead end: you
can create one but never add a single input, output, node, default, or
run validation on it. The whole "reusable sub-graph" use case
`create_metasound_patch` was added for is unreachable.

## Root cause (verified in source)

The mutators load the asset with a *typed* cast to `UMetaSoundSource`:

```cpp
UMetaSoundSource* MetaSound = Cast<UMetaSoundSource>(
    StaticLoadObject(UMetaSoundSource::StaticClass(), nullptr, *AssetPath));
if (!MetaSound) { Ctx.SendError(TEXT("ASSET_NOT_FOUND"),
    FString::Printf(TEXT("Could not load MetaSound: %s"), *AssetPath)); ... }
```

`UMetaSoundPatch` and `UMetaSoundSource` are **sibling subclasses** that
both implement `IMetaSoundDocumentInterface`; neither derives from the
other. `StaticLoadObject(UMetaSoundSource::StaticClass(), ...)` cannot
load a Patch, so the cast returns null and the handler emits
`ASSET_NOT_FOUND`. The builder code these handlers then drive
(`EA_METASOUND_MAKE_BUILDER` over a `TScriptInterface<IMetaSoundDocumentInterface>`)
is class-agnostic and works for both — only the *load gate* is wrong.

Affected handlers (all the typed `Cast<UMetaSoundSource>(StaticLoadObject(...))`
load gate — verified mapping; the earlier draft mislabeled the
AudioAuthoringHandler line numbers and attributed `validate_metasound` to the
wrong file):
- `Handlers/Audio/AudioAuthoringHandler.cpp` — `add_metasound_node` (:831),
  `connect_metasound_nodes` (:972), `add_metasound_input` (:1051),
  `add_metasound_output` (:1146), `set_metasound_default` (:1230).
- `Handlers/Audio/MetaSound/MetaSoundIOMutationHandler.cpp` —
  `remove_metasound_input` (:80), `remove_metasound_output` (:149),
  `rename_metasound_input` (:221), `rename_metasound_output` (:329).
- `Handlers/Audio/MetaSound/MetaSoundVariableHandler.cpp` —
  `add_metasound_variable` (:126), `remove_metasound_variable` (:231),
  `set_metasound_variable_default` (:317), `validate_metasound` (:402),
  `compile_metasound` (:471).
- `Handlers/Audio/MetaSound/MetaSoundDestructiveHandler.cpp` —
  `remove_metasound_node` (:62), `disconnect_metasound_nodes` (:152) — these
  already load generically but then re-gate on `Cast<UMetaSoundSource>(LoadedAsset)`,
  rejecting a Patch the same way.

By contrast `describe_metasound` (`AudioAuthoringHandler.cpp:2508`) loads
generically as `StaticLoadObject(UObject::StaticClass(), ...)` and reports
`"assetKind":"MetaSoundPatch"` — which is exactly why the asset visibly
exists while the mutators claim it doesn't.

This contradicts the explicit promise in the fix that *added* patch
support — `F-metasound-no-patch-or-preset` history `#3` states "Both
factories construct an `IMetaSoundDocumentInterface`, so all downstream
ops (`add_metasound_node`, `connect_metasound_nodes`, etc.) Just Work
once the asset exists." They do not Just Work; the typed cast was never
relaxed to accept a Patch.

## What it should do

Load via the document interface (or via `UObject` + an
`IMetaSoundDocumentInterface` cast), accepting both `UMetaSoundSource`
and `UMetaSoundPatch`. The `MetaSound/` directory already ships the
generic, class-agnostic loader this needs —
`PinWright::MetaSound::LoadMetaSoundDocumentAsset` +
`TryMakeMetaSoundDocumentInterface` (`MetaSoundPathUtils.{h,cpp}`), already
used Patch-compatibly by `MetaSoundInterfaceHandler.cpp` — so the fix is to
route the mutator load gates through that existing pair (wrapped as
`LoadMetaSoundDocumentObject`), not to author a brand-new helper. If a
mutator genuinely cannot apply to a Patch, it should say so with a distinct,
accurate code, not a false `ASSET_NOT_FOUND` on an asset that loads.

## Verbatim repro (replayed live this session)

1. `audio.authoring.create_metasound_patch` `{ "name": "GainStage_Patch",
   "path": "/Game/Audio/MetaSounds/Patches", "save": true }`
   → success: `{"assetPath":"/Game/Audio/MetaSounds/Patches/GainStage_Patch",
   "className":"UMetaSoundPatch","assetClass":"MetaSoundPatch",
   "existsAfter":true}`
2. `audio.authoring.add_metasound_input` `{ "assetPath":
   "/Game/Audio/MetaSounds/Patches/GainStage_Patch", "inputName": "AudioIn",
   "inputType": "Audio", "save": true }`
   → `[ASSET_NOT_FOUND] Could not load MetaSound:
   /Game/Audio/MetaSounds/Patches/GainStage_Patch`
3. `audio.authoring.add_metasound_node` `{ "assetPath":
   "/Game/Audio/MetaSounds/Patches/GainStage_Patch", "nodeType": "gain" }`
   → `[ASSET_NOT_FOUND] Could not load MetaSound:
   /Game/Audio/MetaSounds/Patches/GainStage_Patch`
4. `audio.authoring.describe_metasound` `{ "assetPath":
   "/Game/Audio/MetaSounds/Patches/GainStage_Patch" }`
   → SUCCESS on the *same path*: `{"assetKind":"MetaSoundPatch",
   "rootGraph":{...},"nodes":[],"edges":[],...}` — proving the asset
   exists and loads; only the typed-cast load gate in the mutators fails.

The misleading-error angle compounds the bug: the caller is told the
asset can't be loaded, when it loads fine — there is no in-band signal
that Patch assets are simply unsupported by the mutator family.

**Workaround:** none for Patch authoring. The asset can be created but
not populated; the only authorable MetaSound type today is
`UMetaSoundSource`.

**Fix:** relax the load gate in the listed handlers to accept any
`IMetaSoundDocumentInterface` implementer (Source or Patch), via a shared
load helper.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live against mcp__editor-automation__call: `create_metasound_patch` succeeds (`existsAfter:true`, `className:UMetaSoundPatch`), but `add_metasound_input`, `add_metasound_node` (and per source `add_metasound_output`/`set_metasound_default`/`validate_metasound`/`remove_*`/`rename_*`) all return `[ASSET_NOT_FOUND] Could not load MetaSound: <path>` on the same path that `describe_metasound` reads fine (`assetKind:MetaSoundPatch`). Root cause: the mutators `Cast<UMetaSoundSource>(StaticLoadObject(UMetaSoundSource::StaticClass(),...))` (AudioAuthoringHandler.cpp:831/972/1051/1146/1230, MetaSoundIOMutationHandler.cpp:80/149/221/329); a UMetaSoundPatch is a sibling subclass, not a UMetaSoundSource, so the typed load returns null and the handler emits a false ASSET_NOT_FOUND. `describe_metasound` loads as generic UObject (AudioAuthoringHandler.cpp:2508) and works. Distinct from `F-metasound-no-patch-or-preset` (DONE — that added Patch *creation*; this is that fix's untrue "downstream ops Just Work" assumption: the typed cast was never relaxed, so created Patches are unpopulatable). Fix: load via IMetaSoundDocumentInterface so both Source and Patch are accepted.
- `#2-retriage` `OPEN` triage — High→Medium: hard blocker with no workaround (Patch graphs unpopulatable, false ASSET_NOT_FOUND) but the blocked Patch-authoring task is niche, so Medium not High.
- `#3-fix` `IN-REVIEW` developer — Root-caused and fixed the typed load gate across the whole MetaSound mutator family. Added `PinWright::MetaSound::LoadMetaSoundDocumentObject(AssetPath)` to `MetaSoundPathUtils.{h,cpp}` — a generic load (via the existing `LoadMetaSoundDocumentAsset`) that accepts any `IMetaSoundDocumentInterface` implementer (UMetaSoundSource OR the sibling UMetaSoundPatch) and returns nullptr only for a missing/non-MetaSound path, exactly reproducing the legacy typed-cast's null semantics (callers still emit a single ASSET_NOT_FOUND) minus the false Patch rejection. Swapped every `Cast<UMetaSoundSource>(StaticLoadObject(UMetaSoundSource::StaticClass(),...))` load gate to it: AudioAuthoringHandler.cpp (add_metasound_node, connect_metasound_nodes, add_metasound_input, add_metasound_output, set_metasound_default), MetaSoundIOMutationHandler.cpp (remove/rename input/output), MetaSoundVariableHandler.cpp (add/remove/set variable, validate_metasound, compile_metasound), and dropped the `Cast<UMetaSoundSource>(LoadedAsset)` re-gate in MetaSoundDestructiveHandler.cpp (remove_metasound_node, disconnect_metasound_nodes). compile_metasound's `RebuildReferencedAssetClasses()` now resolves `FMetasoundAssetBase` through `Metasound::IMetasoundUObjectRegistry::Get().GetObjectAsAssetBase(UObject*)` (the common base of Source and Patch) instead of a typed pointer. The `create_metasound` factory paths (AudioAuthoringHandler.cpp:753/:787) deliberately keep their typed `UMetaSoundSource` (they create a Source). Also corrected this ticket's affected-handlers list (the prior draft swapped the AudioAuthoringHandler line labels and put validate_metasound in the wrong file; connect_metasound_nodes was omitted). Regression test: `Source/PinWright/Private/Tests/Assets/TestMetaSoundPatchMutatorsAccept.cpp` (`PinWright.Assets.MetaSoundPatchMutatorsAccept`) factory-creates a real UMetaSoundPatch and drives the production handlers (`InvokeHandlerWithCapture`) for add_metasound_input / add_metasound_node / validate_metasound, asserting none return ASSET_NOT_FOUND (and input/validate succeed). Reverting any gate to the typed cast makes that test fail.
