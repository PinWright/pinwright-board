---
id: B-add-instances-auto-foliage-type-name-mismatch
title: "The auto-created UFoliageType's package stem disagrees with its object name (package Auto_X, object X), so its package handle resolves nowhere: foliage.paint returns that unresolvable form on every call after the first, asset.exists/asset.save/foliage.remove answer it with false or ASSET_NOT_FOUND, and foliage.get_instances answers it with success:true and an empty list over eight live instances"
status: OPEN
severity: High
category: bug
tags: [foliage, add_instances, paint, auto-create, foliage-type, object-path, package-path, name-mismatch, asset-not-found, false-negative, silent-failure, save, no-disk-write, mcp-safe-asset-save]
encounters: 2
lastSeen: 2026-08-29T00:00:00+05:00
---

# The auto-created `UFoliageType` is named one thing and packaged as another

Hand `foliage.add_instances` or `foliage.paint` a **static-mesh** path and it
auto-creates a `UFoliageType_InstancedStaticMesh` for you. The package it creates
is named `Auto_<Mesh>`; the object inside it is named `<Mesh>`. Every other asset
in the engine follows `Package.PackageStem`, so the object path is
`/Game/Foliage/Auto_<Mesh>.<Mesh>` and the **package handle**
`/Game/Foliage/Auto_<Mesh>` — the form the content browser shows, the form
`asset.list` rows carry, the form a caller writes by hand — resolves to nothing:
every lookup normalises it to `Auto_<Mesh>.Auto_<Mesh>`, which does not exist.

The instances are placed correctly and the type asset itself is well-formed. What
is broken is the **address**, and the four downstream failures it causes are
individually plausible enough that none of the three zone agents that hit them
identified the cause from a single symptom.

## Mechanism (re-derived, `Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp`, now 1611 lines)

`foliage.add_instances` — the auto-create branch fires only when `foliageTypePath`
fails to load as a `UFoliageType` (`:1179-1181`) but does load as a `UStaticMesh`
(`:1184-1186`):

```cpp
:1188      FString BaseName = FPaths::GetBaseFilename(FoliageTypePath);
:1189      FString AutoFTPath = FString::Printf(TEXT("/Game/Foliage/Auto_%s"), *BaseName);
:1190      UPackage *FTPackage = CreatePackage(*AutoFTPath);
:1191      UFoliageType_InstancedStaticMesh *AutoFT = NewObject<UFoliageType_InstancedStaticMesh>(
:1192          FTPackage, FName(*BaseName), RF_Public | RF_Standalone);
```

`:1189` puts `Auto_` on the **package**. `:1191-1192` names the **object**
`BaseName` — no prefix. The two disagree by exactly the literal `Auto_`. `:1197`
calls `McpSafeAssetSave(AutoFT)` (second half of this ticket, below), and `:1199`
publishes `AutoFT->GetPathName()`, reaching the wire at `:1265`.

`foliage.paint` repeats the split verbatim: package at `:372`, object at
`:385-386`, `McpSafeAssetSave` at `:391`, `GetPathName()` at `:393`, wire at `:545`.

**A fix must change both sites in one commit** or the two verbs disagree about what
the same mesh's auto-type is called.

For contrast, the two auto-create sites that are **correct** are `foliage.add_type`
(package `FullPackagePath` from `AssetName` at `:930-933`, object `FName(*AssetName)`
at `:947-948`) and `foliage.create_procedural` (package and object both from `FTName`,
`:1438-1445`). So this is a two-site slip, not a house convention.

## `foliage.paint` returns two different strings for the same asset

`paint` — and only `paint` — has an already-exists branch (`:374-381`):

```cpp
:375      if (UEditorAssetLibrary::DoesAssetExist(AutoFTPath)) {
:376        FoliageType = LoadObject<UFoliageType>(nullptr, *AutoFTPath);
:377        if (FoliageType) {
:378          FoliageTypePath = AutoFTPath;          // package-only. Does not resolve.
```

So `paint`'s **first** call on a given mesh returns `GetPathName()` at `:393` — the
dotted form, which resolves — and **every later call** returns `AutoFTPath` at
`:378`, the package-only form, which does not. Same verb, same mesh, same asset,
two different strings, only one of them usable, and nothing in the response says
which one you got.

`foliage.add_instances` has **no** already-exists branch — there is no
`DoesAssetExist` anywhere in `:1176-1206` — so it always returns the dotted form
and its own echo is usable. The package handle is still poison for it: it is what
every other path-taking verb is given when the caller reads the asset's name from
anywhere other than that one response field.

## Four measured consequences, four different verbs

All four were observed live this session by three independent zone agents building
vegetation on this map; the reports were pooled, so a given consequence cannot be
attributed to a named agent. Each is an actual call and an actual response, not a
reconstruction. All four were made against the **bare package handle**
`/Game/Foliage/Auto_SM_MT_Boul_A`; the pooled reports do not record whether each
agent got that string from `paint`'s repeat-call echo (`:378`/`:545`) or wrote it
from the package name — both are reachable in a normal session, and the fix is the
same either way.

**1 — `asset.exists` returns `false` for a package the level actively references.**
`asset.exists` is a bare registry probe: `UEditorAssetLibrary::DoesAssetExist(AssetPath)`
(`Handlers/Asset/AssetManageHandler.cpp:948` registration, `:956` body). The registry
holds the object at `/Game/Foliage/Auto_X.X`; the package handle normalises to
`.Auto_X` and misses. A caller checking whether its own auto-created type exists is
told it does not.

**2 — the asset was absent from disk, and neither save verb could reach it.**
`EditorAssetLibrary.save_asset` on the package handle is a silent no-op, and
`asset.save` on it returns `ASSET_NOT_FOUND`: its resolution is
`LoadObject<UObject>(nullptr, *AssetPath)` (`Handlers/Asset/AssetSaveHandler.cpp:57-63`),
which cannot find an object named `Auto_X` inside package `Auto_X`. The `.uasset`
was genuinely **not on disk** until an explicit `save_loaded_asset` against the
already-loaded object. Why it was not on disk is the separate second half below —
but the name mismatch is what blocked the obvious repair, and made the two defects
corroborate each other's wrong diagnosis.

**3 — `foliage.remove` will not resolve the package handle.** It resolves with
`LoadObject<UFoliageType>(nullptr, *FoliageTypePath)` at `:649` and hard-refuses a
null at `:650-654` with `ASSET_NOT_FOUND`. Measured:
`/Game/Foliage/Auto_SM_MT_Boul_A` → `ASSET_NOT_FOUND`;
`/Game/Foliage/Auto_SM_MT_Boul_A.SM_MT_Boul_A` → works. The verb's bare-name
convenience makes it worse: `:618-623` expands a name with no `/` to
`/Game/Foliage/<name>`, so `Auto_SM_MT_Boul_A` is *accepted as a path* and then
fails to load, and the error reads like a missing asset.

**4 — `foliage.get_instances` returns `success:true` with an empty list over eight
live instances.** The worst of the four: a false negative dressed as a success. The
filtered branch gates on the same registry probe (`:779`) and, when it misses, takes
an early return that reports an empty result as a clean read:

```cpp
:779    if (!UEditorAssetLibrary::DoesAssetExist(FoliageTypePath)) {
:780      // If asked for a specific type that doesn't exist, return empty list gracefully
:781      TSharedPtr<FJsonObject> Resp = MakeShared<FJsonObject>();
:782      Resp->SetBoolField(TEXT("success"), true);
:783      Resp->SetArrayField(TEXT("instances"), TArray<TSharedPtr<FJsonValue>>());
:784      Ctx.SendSuccess(Resp);
:785      return true;
```

Measured: `{foliageTypePath:"/Game/Foliage/Auto_<Mesh>"}` → `{success:true, instances:[]}`;
the dotted `Auto_<Mesh>.<Mesh>` form → all eight. The only tell that the two responses
differ in kind is what the early return **omits**: the normal path always emits
`count`, `orphanedInstanceCount`, `foliageActorPath` and `existsAfter` (`:829-840`),
and this branch emits none of them. A caller reading `instances` — the documented
field — sees a healthy zero and concludes its scatter never happened.

**The pattern is what makes this expensive.** Three of the four fail *loudly but
wrongly*: `ASSET_NOT_FOUND` on an asset that exists and is loaded, because the
failure is in the **path shape**, not the asset. That code sends the reader hunting
a missing or unsaved asset — the wrong hypothesis, which the second half of this
ticket then happens to confirm. The fourth fails silently. There is no error code in
the plugin that means *this asset exists but you addressed it wrong*.

## Second half, separable: the documented auto-create does not persist

`McpSafeAssetSave` is at `Utils/AssetUtils.cpp:362-377` and its entire body is:

```cpp
:375    Asset->MarkPackageDirty();
:376    FAssetRegistryModule::AssetCreated(Asset);
```

under the comment at `:367-369`: *"UE 5.7+ Fix: Do not immediately save newly
created assets to disk. Saving immediately causes bulkdata corruption and crashes."*
The header restates it (`Utils/AssetUtils.h:102-104`: *"It deliberately does NOT
write the `.uasset` … so NOTHING it does makes an edit durable"*). Both foliage
auto-create sites route through it (`FoliageHandler.cpp:1197`, `:391`), and
`add_instances` then reports `existsAfter: true` as a hardcoded literal at `:1266`
with no disk probe. Two zones had to save the asset explicitly.

`B-foliage-paint-does-no-ground-projection` § "Side effect worth knowing about"
(`:85-87`) states the opposite as fact:

> Given a `foliageTypePath` that loads as a `UStaticMesh` rather than a `UFoliageType`, the verb
> **creates and saves a new asset**: `/Game/Foliage/Auto_<MeshBaseName>` …

That sentence is **false** — the verb creates and marks dirty. Recorded here as a
contradicted claim, not edited there.

`E-foliage-add-type-auto-save-undocumented` `#2` is the ground truth on
`McpSafeAssetSave` and explicitly routes this residue:

> The only genuine phenomenon … is the INVERSE of this ticket and is already an owned
> defect class (`B-niagara-save-no-disk-write`, `B-metasound-create-save-no-disk-write`,
> `B-audio-create-save-no-disk-write`); it belongs there, not as a reword of this docs ticket.

**No foliage member of that family was ever filed.** This section is it.

The two halves are separable and should stay separately verifiable: the name
mismatch is board-wide unique, the missing save is a new member of an existing
family with a shipped fix pattern.

## Board hygiene: do not reuse the foliage tickets' line numbers

Citations across the foliage tickets have drifted badly and were re-derived for this
one. `E-foliage-remove-silent-edge-inputs` cites the `foliage.remove` body as
`:303-329` (now `:670-701`); `B-foliage-mutators-no-transaction` calls
`FoliageHandler.cpp` 1233 lines (now 1611);
`B-foliage-paint-does-no-ground-projection` cites the paint auto-create as `:170-200`
with density at `:193` (now `:371-395`, density at `:389`);
`E-foliage-add-type-auto-save-undocumented` `#2` cites `FoliageHandler.cpp:592` for
the `add_type` save (now `:961`) and `AssetUtils.cpp:214-226` for `McpSafeAssetSave`
(now `:362-377`). None of that changes their reasoning; all of it will mislead a
fixer who greps by number.

## Same shape as

Not an umbrella — these are distinct defects that happen to share the two
construction sites at `FoliageHandler.cpp:385-386` and `:1191-1192`:

- `B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) — the sibling verb's
  own defect, and the source of the contradicted "saves" sentence quoted above.
- `F-ism-per-instance-transforms` (IN-REVIEW, High) — names this exact construction
  site and treats the auto-created type as a **wrong destination**: a caller who
  wanted per-instance transforms on a HISM gets a parallel foliage scatter instead.
  Different defect, same code; a fixer touching `:1188-1200` will meet both.
- `B-ground-instances-default-component-foreign-scatter` — a third consequence of the
  same lines: because the auto-type is keyed on the **mesh basename alone**
  (`FPaths::GetBaseFilename`, `:1188` / `:371`), two agents scattering the same mesh
  in a multi-agent level land in one shared component. Filed in parallel this session;
  the id is reserved.
- `B-audio-create-save-no-disk-write` (IN-REVIEW, Critical) — family head for the
  second half, with the fix pattern to borrow.
- `E-foliage-add-type-auto-save-undocumented` (WONTFIX, Low) — the routing decision
  this ticket executes.

## Severity

**High.** Impact class is *silent false-success / silent wrong data on a normal path*:
consequence 4 returns `success:true` with an empty `instances[]` over eight live
instances, and a caller that branches on emptiness concludes its scatter did not
happen and re-runs it. The other three are hard blockers whose workaround only
exists once you know the cause — `Medium` on their own; the false negative sets the
band. `paint` handing back two different strings for one asset depending on call
count is the same class: the second string is silently wrong data on a normal path.

**Not Critical.** Nothing is corrupted or lost. The `UFoliageType` is well-formed,
its instances are intact in the `AInstancedFoliageActor`, and every one of the four
failures is fully recovered by passing the `Package.AssetName` form — no editor
crash, no destroyed data, no unrecoverable state. The object/package split is a
naming error in a returned string, not a bad write.

**Reach modifier declined (no bump to Critical).** `foliage.add_instances` is the
documented literal-placement verb (`Docs/wiki-src/foliage.md:122`,
`level-building.md:84`) so any vegetation workflow reaches it, but the auto-create
fires **only** on the static-mesh-path input branch: a caller passing a real
`UFoliageType` path loads at `:1179-1181` and never enters it. That is the common
convenience form, not an almost-every-session path, so it earns neither the up-bump
to Critical nor the down-bump to Medium. Holds at High on impact alone.

## Fix

**Name the object to match the package stem.** One line at each of the two sites:

```cpp
FName(*FString::Printf(TEXT("Auto_%s"), *BaseName))   // FoliageHandler.cpp:1192 and :386
```

Both must change **in one commit** — `add_instances` and `paint` must agree about
what a given mesh's auto-type is called, or a level painted by one is unaddressable
by the other.

**This changes the path of every already-created `Auto_*` asset.** Anything on disk
or in a level today is `Auto_X.X`; after the fix the code creates and looks up
`Auto_X.Auto_X`, and `paint`'s already-exists check (`:375-381`) will miss the old
one and create a second type beside it. That needs either a migration pass or an
explicit decision that existing `Auto_*` assets keep the broken form and are
addressed the long way. **The migration question was not investigated.**

**Separately, make the resolution paths accept a bare package path.** This is the
`Package.AssetName` retry that `agent-conventions.md:116` already documents as a
known trap:

> `StaticFindObject` on a bare long-package path (`/Game/Foo/Bar`) matches the outer
> `UPackage`, not the asset — non-null result + failed `Cast<T>` is the symptom; retry
> the `Package.AssetName` form (`MetaSoundPathUtils.cpp::LoadMetaSoundDocumentAsset`).

Same trap, different surface: here the first lookup returns *null* rather than a
`UPackage`, but the retry is identical. Apply it at `foliage.remove:649`,
`foliage.get_instances:779` and `:788-789`, and the two type loads (`:1179-1181`,
`:362-363`). Worth doing **even after** the rename, because it is what fixes the
already-created assets without a migration.

**And make `paint` stop publishing the unresolvable form.** `:378` should assign
`FoliageType->GetPathName()`, not `AutoFTPath`, so the first and subsequent calls
return the same string. That is a one-line fix independent of the rename and worth
landing whichever way the migration question goes.

**And make `get_instances`'s miss branch honest.** `:779-785` should not report a
registry miss as a clean empty read. Either error (the caller named a type that does
not resolve) or emit the same field set as the normal path with a flag saying the
filter matched no known type — the present response is indistinguishable from a real
zero, which is what cost the session.

**For the second half**, borrow the shipped pattern from `B-audio-create-save-no-disk-write` `#2`:
route the auto-create through `SaveAssetToDiskReportingPresence` (`AssetUtils.h:241`,
which forces `SaveLoadedAssetThrottled` and probes with `IFileManager::FileSize` behind
`ShouldTreatAssetSaveAsSuccess`), or — if the create is deliberately left deferred —
report it with `AddMarkDirtySaveReport` (`AssetUtils.h:293`, which emits a measured
`{saveRequested, saved, pendingFlush, markedForSave}`) instead of the hardcoded
`existsAfter:true` at `:1266`. A `UFoliageType_InstancedStaticMesh` is not a
Blueprint/SCS asset, so the bulkdata-corruption vector that forced `McpSafeAssetSave`
does not apply, exactly as argued for the audio assets.

## Workaround

Never address an auto-created type by its package handle. Use
`/Game/Foliage/Auto_<Mesh>.<Mesh>` — all four verbs above work given that form.
For `foliage.paint` specifically, ignore the `foliageTypePath` it echoes (it is the
usable dotted form only on the first call for a given mesh) and rebuild the dotted
string yourself. Then save the asset explicitly against the loaded object; nothing
in any response tells you that is required.

## History
- `#1-name-mismatch-and-no-save` `OPEN` reporter — Filed from live vegetation work on this map; two zones hit it independently (`encounters: 2`), four consequences reported by three zone agents in one session, pooled so no consequence is attributable to a named agent. Root cause: both foliage auto-create sites build a package named `/Game/Foliage/Auto_<Mesh>` but name the object `<Mesh>` with no `Auto_` prefix — `foliage.add_instances` at `FoliageHandler.cpp:1189` (package) / `:1191-1192` (object), `foliage.paint` at `:372` / `:385-386` — so the object path is `Auto_X.X` and the package handle `Auto_X` normalises to `Auto_X.Auto_X` and resolves to nothing. `foliage.add_type` (`:930-933`/`:947-948`) and `foliage.create_procedural` (`:1438-1445`) name package and object consistently, so this is a two-site slip, not a convention. `foliage.paint` compounds it: it is the only one of the two with an already-exists branch (`:374-381`), so its FIRST call on a mesh returns `GetPathName()` (`:393`, dotted, resolves) and every LATER call returns `AutoFTPath` (`:378`, package-only, does not) — same verb, same asset, two different strings, echoed at `:545` with nothing saying which you got. `add_instances` has no such branch (no `DoesAssetExist` anywhere in `:1176-1206`) and always echoes the dotted form at `:1199`/`:1265`. Four measured downstream failures, all against the bare package handle `/Game/Foliage/Auto_SM_MT_Boul_A` (the pooled reports do not record whether each agent got it from paint's repeat echo or wrote it from the package name): (1) `asset.exists` returns false (`AssetManageHandler.cpp:948` registration, `:956` bare `DoesAssetExist`); (2) `EditorAssetLibrary.save_asset` silently no-ops and `asset.save` returns `ASSET_NOT_FOUND` (`AssetSaveHandler.cpp:57-63`), with the `.uasset` genuinely absent from disk until an explicit `save_loaded_asset` on the loaded object; (3) `foliage.remove` refuses the handle with `ASSET_NOT_FOUND` (`:649`, `:650-654`) and accepts only `Auto_SM_MT_Boul_A.SM_MT_Boul_A`, made worse by the bare-name expansion at `:618-623` that accepts the handle as a path before failing to load it; (4) worst — `foliage.get_instances` returns `{success:true, instances:[]}` over eight live instances via the early return at `:779-785`, a false negative dressed as success, distinguishable from a real zero only by the fields it omits (`count`/`orphanedInstanceCount`/`foliageActorPath`/`existsAfter`, all present on the normal path at `:829-840`). Three fail loudly but wrongly — `ASSET_NOT_FOUND` names the wrong thing, since the asset exists and is loaded and only the path shape is wrong — and the fourth fails silently. Second, separable half: the auto-create does not persist. `McpSafeAssetSave` (`Utils/AssetUtils.cpp:362-377`) is `MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()` and nothing else, under the UE 5.7+ bulkdata-corruption comment (`:367-369`; header `AssetUtils.h:102-104`), yet `add_instances` reports a hardcoded `existsAfter:true` (`:1266`). This makes `B-foliage-paint-does-no-ground-projection:85-87` ("**creates and saves a new asset**") false — quoted, not edited — and fills the foliage gap `E-foliage-add-type-auto-save-undocumented` `#2` routed to the `B-niagara`/`B-metasound`/`B-audio` no-disk-write family, which had no foliage member. The two defects corroborated each other's wrong diagnosis: a not-found code on a path-shape error looks exactly like the missing asset the absent disk write independently produced. Cross-links verified against the board: `F-ism-per-instance-transforms` (IN-REVIEW/High, same construction site as a wrong-destination defect), `B-ground-instances-default-component-foreign-scatter` (parallel filing this session, id reserved, third consequence of the same lines via the mesh-basename-only key at `:1188`/`:371`), `B-audio-create-save-no-disk-write` (IN-REVIEW/Critical, fix pattern). Severity High: impact class is silent false-success on a normal path (consequence 4, plus paint's second string); declined Critical because nothing is corrupted or lost and all four are fully recovered by the `Package.AssetName` form; declined the reach up-bump because the auto-create fires only on the static-mesh-path branch (`:1179-1181` short-circuits a real `UFoliageType` path), so it is a common convenience form rather than an almost-every-session path. Proposed fix: prefix the object name at both `:1192` and `:386` in one commit; make `paint:378` publish `GetPathName()`; add a `Package.AssetName` fallback at the four resolution sites (the `agent-conventions.md:116` `StaticFindObject` trap, same shape); make the `get_instances` miss branch honest. **No fix was attempted, and the migration cost of renaming already-created `Auto_*` assets was not investigated** — the rename changes every existing asset's path and would make `paint`'s already-exists check (`:375-381`) create a duplicate beside the old one. Every line number here was re-derived against the current 1611-line `FoliageHandler.cpp`; the existing foliage tickets' numbers have drifted by hundreds of lines and must not be reused.
