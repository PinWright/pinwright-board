---
id: F-generic-asset-create-verb
title: "40 asset.* verbs and not one creates an asset from a caller-named class, so any asset type without a bespoke create verb is unreachable — UProceduralVegetation above all, whose inner graph is built by a non-UFUNCTION behind a private UPROPERTY, closing the python.execute escape hatch as well"
status: OPEN
severity: Medium
category: feature
tags: [asset, asset-create, generic-creator, factory, class-parameter, missing-verb, procedural-vegetation, modal-hazard, python-unreachable]
encounters: 1
lastSeen: 2026-08-29T17:35:00+05:00
---

# Asset creation in this plugin is per-type, and the type list is closed

`grep -rn 'REGISTER_RPC_HANDLER("asset.' Source/` returns **40** verbs. None of them creates an
asset from a class the caller names.

The three nearest misses, so the gap is not assumed:

- **`asset.import`** (`Handlers/Asset/AssetManageHandler.cpp:179`) is a factory path, but the
  factory is *"picked by extension"* (FBX, PNG, WAV) — it needs a file on disk, and the caller
  cannot name a class.
- **`asset.duplicate`** (`:308`) needs an existing source asset of the type you want.
- **`asset.create_folder`** (`:904`) does not create an asset.

Everything else that creates an asset in this plugin is bespoke and per-namespace:
`pcg.create_graph`, `data_table.create`, the 22 typed `volume.create_*` verbs, and so on. That is a
closed enumeration, and an asset class nobody wrote a verb for has no route.

## The forcing case: the Procedural Vegetation editor's asset

This session made PV *graphs* creatable for the first time (`F-pcg-create-graph-class-parameter`,
DONE). It did not make PV usable, because the asset the PV editor actually opens is a different
object.

**1. The editor's asset is `UProceduralVegetation`, not a `UPCGGraph`.**
`UAssetDefinition_ProceduralVegetation::GetAssetClass()` returns `UProceduralVegetation::StaticClass()`
— `C:/UE_5.8/Engine/Plugins/Experimental/ProceduralVegetationEditor/Source/ProceduralVegetationEditor/Private/AssetDefinition_ProceduralVegetation.cpp:21-24`
(declared `AssetDefinition_ProceduralVegetation.h:16`).

**2. It owns the graph; it is not one.**
`.../ProceduralVegetation/Public/ProceduralVegetation.h`:

```cpp
UCLASS(Abstract)                                                            // :47
class PROCEDURALVEGETATION_API UProceduralVegetationInterface : public UObject   // :48
...
UCLASS()                                                                    // :60
class PROCEDURALVEGETATION_API UProceduralVegetation : public UProceduralVegetationInterface  // :61
{
    UPROPERTY()                                                             // :65
    TObjectPtr<UProceduralVegetationGraph> Graph;                           // :66
```

**Correction to the lead this was filed from:** `UProceduralVegetation` is *not* a plain `UObject`.
It derives from the abstract `UProceduralVegetationInterface` (`:47-48`), which derives from
`UObject`. That matters for the ask — a generic creator has to accept "any `UObject` subclass a
factory can build", not a narrower notion. There is also a sibling implementation,
`UProceduralVegetationInstance` (`:92`), which owns a `UProceduralVegetationGraphInstance` instead,
so "owns an inner graph" is specific to this class rather than to the family.

**3. The inner graph is designed to be inner.** `UProceduralVegetationGraph`'s inline constructor
sets `bIsStandaloneGraph = false` (`ProceduralVegetation.h:19`) and the class hard-refuses to change
it: `virtual bool CanToggleStandaloneGraph() const override { return false; };` (`:34`). So
`pcg.create_graph`'s product — a correct, standalone `UProceduralVegetationGraph`, verified on disk
this session — is by design *not* the thing the editor opens, and `pcg.create_graph`'s
`IsChildOf(UPCGGraph)` gate (`PCGGraphCreate.cpp:62`) structurally excludes the outer asset.
**That gate should stay.** Widening it would turn a graph-creation verb into a general asset
factory by the back door, which is this ticket's ask and does not belong there.

**4. `python.execute` cannot substitute, and this is the load-bearing negative.** The inner graph is
built by `UProceduralVegetation::CreateGraph` (`.../Private/ProceduralVegetation.cpp:22-33` — note
the range: the reported `:22-32` truncates the closing brace), which either duplicates a template
graph or calls `NewObject<UProceduralVegetationGraph>(this, PV::DefaultGraphName, RF_Transactional)`.
That function carries **no `UFUNCTION`** (`ProceduralVegetation.h:73`; neither does `SetGraph` at
`:72`), and the class's only reflected member is the `Graph` `UPROPERTY` at `:65-66`, which is
**private** — the `public:` does not begin until `:68`. Its only in-tree callers are C++
(`ProceduralVegetationFactory.cpp:29` and `:35`).

So a Python caller can `new_object` a `UProceduralVegetation` and will get one with a **null inner
graph**, with nothing reachable through reflection able to populate it. The escape hatch this board
normally treats as the fallback is closed for this class specifically.

**5. And there is no PV editing session without one.** `FPVEditor` is constructed in exactly one
place in the whole plugin — `MakeShared<FPVEditor>()` at
`AssetDefinition_ProceduralVegetation.cpp:47`, inside `OpenAssets` (`:32-52`). (Caveat worth
carrying: `:40-45` short-circuits before that for an embedded subgraph, handing off to
`UAssetEditorSubsystem::OpenEditorForAsset` and constructing no `FPVEditor`, so "opening a PV asset
always builds an `FPVEditor`" is not quite true — but every path that does build one starts from a
`UProceduralVegetation`.)

## Modal hazard — not a defect today, but a precondition on the fix

`UProceduralVegetationFactory::ConfigureProperties()`
(`.../ProceduralVegetationEditor/Private/ProceduralVegetationFactory.cpp:42-57`, declared
`ProceduralVegetationFactory.h:22`) opens a blocking modal at `:46-49`:

```cpp
const bool bPressedOk = SPVCreateDialog::OpenCreateModal(
    LOCTEXT("ProceduralVegetationFactoryConfigureProperties", "Create Procedural Vegetation"),
    SampleProceduralVegetation
);
```

`SPVCreateDialog::OpenCreateModal` (`Private/Widgets/SPVCreateDialog.cpp:128`, declared
`SPVCreateDialog.h:29`) reaches `GEditor->EditorAddModalWindow(CreateWindow)` at `:142`, and
`UEditorEngine::EditorAddModalWindow` ends in `FSlateApplication::Get().AddModalWindow(...)`
(`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/EditorEngine.cpp:4172`).

That is exactly the shape `B-physics-asset-factory-modal-hang` documents. **It was not invoked in
this session** — no factory-driven creation was attempted, and this is recorded as a hazard, not a
sighting. If a generic factory-driven creation verb is built, PV must be on the modal-bypass list.
And that ticket's own `#2` is the reason a bypass cannot be assumed rather than verified: under
`GIsRunningUnattendedScript`, `UPhysicsAssetFactory` leaves `NewBodyResponse` unset and returns
nullptr, *converting the hang into a silent no-asset failure*. Whichever mechanism a generic verb
uses, each factory it can reach needs checking against both failure modes.

This note **should** have been appended to `B-physics-asset-factory-modal-hang` itself. It could
not be: that file carries an uncommitted edit by another host (`git status` ` M`, an in-flight `#3`
entry about the dispatcher's `FScopedUnattendedRpc` guard), and committing it would have swept a
sibling's work into an unrelated commit. Recorded here instead, on the ticket a fixer of the generic
verb will actually be reading; whoever next touches that file should carry it across.

## The ask

    asset.create(className, name, savePath, [factoryClass])

- Resolve `className` through `Utils/ClassUtils::ResolveUClass` — the plugin's mandated three-tier
  resolver, and the same choice `pcg.create_graph`'s `graphClass` made deliberately so that no
  compile-time link dependency on an optional module is created.
- Create through `IAssetTools::CreateAsset` with a factory resolved from the class, which is the
  pattern `data_table.create` already ships (`F-data-table-create-asset` `#2`: loads and gates the
  row struct, then `IAssetTools::CreateAsset` with a pre-seeded `UDataTableFactory`).
- Handle `ConfigureProperties` modals explicitly rather than hoping — see the hazard section — and
  return the resolvable **object** path so the create → configure loop works off the response, as
  `data_table.create` does.
- Guard `ASSET_EXISTS`, `MarkPackageDirty`, no auto-save; the house pattern.

**This does not propose deleting the typed verbs.** They carry class-specific defaults, validation
and echoes a generic verb cannot — `data_table.create` gates its row struct, `pcg.create_graph`
gates `IsChildOf(UPCGGraph)` and rejects abstract classes. The ask is a **floor under the classes
nobody enumerated**, not a replacement for the ones somebody did. Same position
`F-generic-volume-creator-with-brush-geometry` takes on its own 22 typed verbs.

## Why this is filed as one ticket and not three, and with no umbrella above it

The session's headline finding was *"PV is authorable through PinWright but not usable"*, for three
independent structural reasons. Filing that sentence would be an umbrella. It decomposes cleanly,
and two of the three parts already have the right home:

- **(a) Wrong asset — this ticket.** Concrete code site, concrete ask, testable without PV
  (`asset.create` on any class with a factory).
- **(b) Export does not export — deliberately NOT on this board.** `FPVExportElement::ExecuteInternal`
  (`.../ProceduralVegetation/Private/Nodes/PVExportSettings.cpp:112-138`) only validates and copies
  input to output; the real writing path runs from `FPVEditor::OnExport()`
  (`.../ProceduralVegetationEditor/Private/PVEditor.cpp:786-834`) through
  `FPVExporter::Export()` (`Private/Exporter/PVExporter.cpp:141`), needs an open PV toolkit, reads
  the editor's own inspection cache (`PVEditor.cpp:476`, fed by a `UPCGDefaultExecutionSource`
  created at `:93` — not a `UPCGComponent`), and is gated behind
  `SPVExportSelectionDialog::ShowModal()` at `:825`. **That is UE 5.8's design, not a PinWright
  defect**, and the board's scope line is MCP tool issues only. Routed to `Docs/map/` per the
  precedent the vegetation dossier's § H set for engine and content findings. Its only in-scope
  residue — that a completed generate through a wired Export node yields a receipt from which
  nothing can be concluded — is not a separate defect; it is (c).
- **(c) Output unobservable — `B-pcg-generate-blind-to-non-point-data`** (OPEN, High, filed this
  session).

And no umbrella above the three, following the position already recorded on this board by
`F-pcg-create-graph-class-parameter` § *Distinct from*: *"No umbrella 'support Procedural
Vegetation' ticket is proposed… An umbrella would be a wish with no repro on a board whose
convention is repro-backed items."* That reasoning has not changed; what changed is that the
decomposition now has one more member and each member has a repro.

## Dedup

Board-wide search for generic asset creation. `F-data-table-create-asset` (IN-REVIEW, Medium) is the
same shape one type down — *"No RPC to create a DataTable asset — `data_table.*` is row-CRUD only,
`asset.*` create is folder/material only"* — and its `#2` confirms the same absence independently
(*"no `asset.*` verb or `UDataTableFactory` created a `UDataTable` anywhere in source"*). That is
corroboration, not duplication: it asked for one typed verb and got one, which is precisely the
per-type pattern this ticket argues has no floor under it. `F-generic-volume-creator-with-brush-geometry`
(OPEN, Medium) is the same argument scoped to `AVolume` subclasses and is the nearest sibling; it
stays separate because its ask is specifically a *brush-geometry-bearing* creator and a generic
`asset.create` would not solve it (a volume is a level actor, not a content-browser asset).
`F-anim-add-graph-node-generic` and `F-rpc-audio-describe-metasound` are unrelated. **No ticket asks
for a generic asset-creation verb.**

## Severity

**Medium, argued.** Impact class is the rubric's *"High or Medium: hard blocker with no workaround (a
stub, a **missing verb**, or rejecting valid input)"*. It lands on **Medium** because for the general
case a workaround exists: `python.execute` reaches `IAssetTools::CreateAsset` and
`unreal.AssetToolsHelpers` for any class whose factory does not need C++-only setup, which is the
rubric's Medium — *"doable, but only via a documented workaround, a source dive, or many extra
calls"*.

**The High reading, stated plainly so a reviewer can take it up, because it is a good argument.**
For the forcing case there is **no** route at all: `CreateGraph` is not a `UFUNCTION` and `Graph` is
a private `UPROPERTY`, so even `python.execute` cannot build a correct `UProceduralVegetation` — it
gets an asset with a null inner graph. A reviewer who scores this ticket by the case that provoked it
lands on High and is not being unreasonable. It is filed Medium because the *ask* is the generic
verb, whose value and whose blocked population are both wider and shallower than the PV instance
inside it, and because rating a capability ask by its worst instance would put it above tickets that
are silently losing data today. If the generic verb is built and PV still cannot be created through
it, that is a follow-on ticket with its own severity, not a reason to inflate this one now.

**Reach modifier declined in both directions, and named.** No bump up: asset creation is common, but
no single un-enumerated class is created in almost every session — the common ones already have
typed verbs, which is exactly why the gap went unnoticed. No bump down: this is not a rare edge path
either, since every new asset type the plugin grows support for arrives here first. Medium stands
unmodified.

## History
- `#1-no-verb-creates-an-asset-from-a-class` `OPEN` reporter — Gap verified by grep: `REGISTER_RPC_HANDLER("asset.` returns 40 verbs across seven handler files and **none** creates an asset from a caller-named class; the nearest are `asset.import` (`AssetManageHandler.cpp:179`, factory picked by file extension), `asset.duplicate` (`:308`, needs a source asset) and `asset.create_folder` (`:904`, not an asset), while every other creation path in the plugin is bespoke and per-namespace (`pcg.create_graph`, `data_table.create`, 22 typed `volume.create_*`). Forcing case, all re-derived against UE 5.8 source: the PV editor's asset is `UProceduralVegetation` (`AssetDefinition_ProceduralVegetation.cpp:21-24`), which **corrects the lead** — it is not a plain `UObject` but derives from the abstract `UProceduralVegetationInterface` (`ProceduralVegetation.h:47-48`, subclass at `:60-61`) — and owns the graph in a **private** `UPROPERTY` at `:65-66` (the `public:` is at `:68`); the inner graph is designed to be inner (`bIsStandaloneGraph = false` at `:19`, and `CanToggleStandaloneGraph()` hard-returns false at `:34`), so `pcg.create_graph`'s correct standalone product is by design not the editor's asset and its `IsChildOf(UPCGGraph)` gate (`PCGGraphCreate.cpp:62`) rightly excludes it. **Load-bearing negative:** `UProceduralVegetation::CreateGraph` (`ProceduralVegetation.cpp:22-33` — the reported `:22-32` truncates the closing brace) carries **no `UFUNCTION`** (`ProceduralVegetation.h:73`, and neither does `SetGraph` at `:72`), and the only reflected member is that private `Graph` property, so `python.execute` cannot build a correct asset either — it yields one with a null inner graph. Only in-tree callers are C++ (`ProceduralVegetationFactory.cpp:29`, `:35`). `FPVEditor` is constructed in exactly one place, `AssetDefinition_ProceduralVegetation.cpp:47` inside `OpenAssets` (`:32-52`), with the embedded-subgraph early-out at `:40-45` recorded as a caveat. **Modal hazard recorded, not a sighting:** `UProceduralVegetationFactory::ConfigureProperties()` (`ProceduralVegetationFactory.cpp:42-57`) opens `SPVCreateDialog::OpenCreateModal` at `:46-49`, which reaches `GEditor->EditorAddModalWindow` at `SPVCreateDialog.cpp:142` and `FSlateApplication::AddModalWindow` at `EditorEngine.cpp:4172` — the `B-physics-asset-factory-modal-hang` shape — and it was **not invoked**; PV must be on the modal-bypass list if a generic factory-driven verb is built, and that ticket's `#2` shows a bypass must be verified per factory because `GIsRunningUnattendedScript` converts the physics hang into a silent no-asset failure. That note belongs on `B-physics-asset-factory-modal-hang` and **could not be appended**: the file carries another host's uncommitted `#3` edit, and committing it would have swept a sibling's in-flight work. Ask: `asset.create(className, name, savePath, [factoryClass])` resolving through `ResolveUClass` and creating via `IAssetTools::CreateAsset` with a factory resolved from the class — the pattern `data_table.create` already ships — returning the resolvable object path, guarding `ASSET_EXISTS`, no auto-save, and handling `ConfigureProperties` modals explicitly. Explicitly does not propose deleting the typed verbs; the ask is a floor under un-enumerated classes, the same position `F-generic-volume-creator-with-brush-geometry` takes. **Records the one-or-three judgment**: the session headline "PV is authorable but not usable" decomposes into (a) this ticket, (b) the export path, which is UE 5.8 design rather than a PinWright defect (`PVExportSettings.cpp:112-138` validates and passes through; the writer runs from `PVEditor.cpp:786-834` through `PVExporter.cpp:141`, needs an open toolkit, reads the editor's own inspection cache at `PVEditor.cpp:476` fed by a `UPCGDefaultExecutionSource` at `:93`, and is gated behind a blocking modal at `:825`) and is therefore routed to `Docs/map/` per the dossier § H precedent, and (c) `B-pcg-generate-blind-to-non-point-data`; with **no umbrella above them**, per the position already recorded on `F-pcg-create-graph-class-parameter` § *Distinct from*. Dedup: `F-data-table-create-asset` independently confirms the same absence and is corroboration rather than duplication — it asked for one typed verb and got one, which is the pattern this ticket says has no floor; `F-generic-volume-creator-with-brush-geometry` is the same argument for level actors and would not be solved by `asset.create`; no ticket asks for a generic asset-creation verb. Severity Medium with the High reading stated in full and refused, because the ask is the generic verb rather than its worst instance.
