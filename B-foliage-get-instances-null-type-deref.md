---
id: B-foliage-get-instances-null-type-deref
title: "foliage.get_instances dereferences the IFA map key, which the engine does not guarantee non-null"
status: IN-REVIEW
severity: Medium
category: bug
tags: [foliage, crash-hardening, engine-contract]
---

# foliage.get_instances dereferences the IFA map key, which the engine does not guarantee non-null

`foliage.get_instances`' unfiltered branch called `Type->GetPathName()` on the key
`AInstancedFoliageActor::ForEachFoliageInfo` hands its lambda. That key comes straight out of
`TMap<TObjectPtr<UFoliageType>, TUniqueObj<FFoliageInfo>> FoliageInfos` and is passed unguarded
(`InOperation(Pair.Key, Pair.Value.Get())`, `InstancedFoliage.cpp:3020`). A null key is a state the
engine explicitly recognises, so the deref was a hard editor crash waiting on the right sequence.

## Is the null key reachable

**In principle yes; through the plugin's own surface, no — an engine-side repair closes the window
before any RPC can observe it.** Evidence, all from UE 5.8 source:

**The state is real and the engine names it.** `AInstancedFoliageActor::CleanupDeletedFoliageType()`
(`InstancedFoliage.cpp:5180`) walks `FoliageInfos` with an iterator, tests `It->Key == nullptr`,
strips the entry's instances and `RemoveCurrent()`s it. `PostLoad` prunes null keys a second time
(`:4552`) behind the MapCheck warning *"Foliage instances for a missing static mesh have been
removed"*, using `while (RemoveAndCopyValue(nullptr, ...))` — a loop, because several entries can
collapse onto a null key at once. The engine would not ship two independent pruners for an
impossible state.

**Two production routes into it.**
1. *Load.* `FoliageInfos` is not a UPROPERTY; `Serialize` writes it directly (`Ar << FoliageInfos`,
   `:4420`) with no guard on archive kind. A level whose foliage type asset is gone deserializes the
   key as null. Closed by the `PostLoad` prune above, which runs before anything outside can see the
   actor.
2. *Force delete.* `ObjectTools::ForceReplaceReferences(nullptr, ...)` reaches the key through that
   same unguarded `Serialize` — `FFindReferencersArchive` finds the IFA (it calls
   `PotentialReferencer->Serialize(*this)` **and** proxies `CallAddReferencedObjects`), so the IFA
   lands in the replacement set and `FArchiveReplaceObjectAndStructPropertyRef` rewrites the key to
   null **in place**. This is deliberate engine behaviour, not an accident: `TSparseSet::Serialize`
   rehashes the set when `Ar.IsModifyingWeakAndStrongReferences() && !Ar.IsSaving()`
   (`SparseSet.h.inl:1788`), exactly the replacement-archive case, and
   `FArchiveReplaceObjectRefBase` sets that flag (`ArchiveReplaceObjectRef.h:197`). So after
   `ForceReplaceReferences` the map genuinely holds a correctly-hashed null key.

**What closes route 2.** Still inside `ForceDeleteObjects`, `DeleteSingleObject` calls
`FAssetRegistryModule::AssetDeleted`, which broadcasts `AssetRemovedEvent` **synchronously**
(`AssetRegistry.cpp:4438`). `FFoliageEditModule::NotifyAssetRemoved` is subscribed to it
unconditionally from `StartupModule`, and it runs `CleanupDeletedFoliageType()` over every
`AInstancedFoliageActor` in memory (`FoliageEditModule.cpp:146`). The FoliageEdit module is loaded
unconditionally by `FEditorModeRegistry::Initialize()` (`EditorModeRegistry.cpp:55`), reached from
`UAssetEditorSubsystem::Initialize`. So on a stock editor host the null key exists only inside
`ForceDeleteObjects`, where no RPC can interleave.

**The reporter's proposed mechanism is correct but inert here.** `AddReferencedObjects` does pass the
key to the mutable overload (`Collector.AddReferencedObject(Pair.Key, This)`, `:5214`), and that
overload does hand GC a writable `UObject*&` (`UObjectGlobals.h:2736-2742`), which
`ProcessReferenceDirectly` will `KillReference` (i.e. `Object = nullptr`) for a Garbage-flagged
target. But that only fires when the GC runs with `EGCOptions::EliminateGarbage`, gated on
`UObjectBaseUtility::IsGarbageEliminationEnabled()` (`GarbageCollection.cpp:5684`) — and **both PDS
checkouts disable it**: `gc.GarbageEliminationEnabled=False` under
`[/Script/Engine.GarbageCollectionSettings]` in `Config/DefaultEngine.ini`
(`unreal-fpv-new:421`, `unreal-fpv:421`). With it off, `MayKill` returns `EKillable::No` for native
ARO origins and the key is never nulled by GC. On a host that leaves the setting at its default
(`True`), this route *would* leave a persistent null key — and unlike route 2 it fires no
`OnAssetRemoved`, so nothing prunes it.

## Why the existing suite stayed green

Not because the state is impossible — because the deref is never reached. `DiscardType`
(`TestFoliagePlacementBehaviour.cpp`) calls `foliage.remove` **before** deleting the asset, and
`foliage.remove`'s scoped branch does `Info->Instances.Empty()` without removing the map entry. The
unguarded `Type->GetPathName()` sat *inside* `for (const FFoliageInstance &Inst : Info.Instances)`,
so a zero-instance entry never touches the key. The green test proves the loop body is not entered,
not that the key is non-null.

**Fix:** guard and skip in the unfiltered branch of `foliage.get_instances`
(`FoliageHandler.cpp`), plus a new `orphanedInstanceCount` response field so the skip is not silent.

## Exposure elsewhere

Swept every plugin site touching `FoliageInfos` / `ForEachFoliageInfo` / `GetFoliageInfos`. This was
the **only** unguarded key deref. `foliage.remove`'s `removeAll` lambda names the parameter `Type`
but reads only `Info`; the world-wide instance count in `foliage.create_procedural` takes the key as
an unnamed parameter; `foliage.paint` and `foliage.add_instances` reach `FFoliageInfo` via
`FindInfo(FoliageType)` with an already-resolved non-null type. No other call site needed a change.

## Regression test

**Not written — an honest one is not possible from the plugin's surface.** Producing the state needs
either the FoliageEdit module's `OnAssetRemoved` subscriber suppressed, a project-wide GC cvar
flipped mid-suite, or a hand-built null-keyed map. `FoliageInfos` is private and the only public
inserter, `AddFoliageInfo(UFoliageType*)`, dereferences its argument
(`FoliageType->UpdateGuid`, `:3034`) so it cannot seed a null key either. An end-to-end
add/paint/force-delete/read test would pass identically before and after the guard, because the
engine repair removes the entry — it would exercise nothing. Filing that as a regression test would
be theatre.

## History
- `#1-null-key-deref-hardened` `IN-REVIEW` developer — "Investigated a null-deref reported but not
  reproduced during test-writing. Verdict: the null key is a genuine engine state (the engine ships
  `CleanupDeletedFoliageType()` plus a `PostLoad` pruner for it) and `ForceReplaceReferences` really
  does rewrite the key in place and rehash — but `FFoliageEditModule::NotifyAssetRemoved` repairs it
  synchronously inside `ForceDeleteObjects`, so no RPC can observe it on a stock host. The
  reporter's GC-nulling mechanism is real but inert in both PDS checkouts, which set
  `gc.GarbageEliminationEnabled=False`. Non-reproduction is fully explained by the test teardown:
  `foliage.remove` empties `Info.Instances` before the delete, and the deref lives inside the
  per-instance loop. Fixed as hardening rather than as a reproduced crash: added `if (!Type)` skip
  in the unfiltered branch of `foliage.get_instances` in
  `Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp`, accumulating
  into a new `orphanedInstanceCount` response field (documented on the handler registration) so
  skipped instances are not dropped silently. Justification for guarding despite the closed window:
  the repair lives in the FoliageEdit editor module, not in this plugin; PinWright targets UE 5.3-5.8
  and arbitrary host projects where `gc.GarbageEliminationEnabled` defaults to `True`; and the cost
  is one pointer test in a read verb whose failure mode is a hard editor crash. NOT COMPILED — an
  automation suite was running against the built DLL, so this needs a build before verification. No
  regression test: see the section above for why one cannot honestly be written."
