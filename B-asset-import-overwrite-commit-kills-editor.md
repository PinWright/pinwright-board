---
id: B-asset-import-overwrite-commit-kills-editor
title: "asset.import over an existing asset kills the editor: FScopedOverwriteStage::Commit calls ObjectTools::ForceReplaceReferences, which walks every UFunction's bytecode and faults with EXCEPTION_ACCESS_VIOLATION reading 0xffffffffffffffff"
status: OPEN
severity: Critical
category: bug
tags: [asset, import, overwrite, objecttools, ForceReplaceReferences, crash, editor-killer, texture, multi-agent, data-loss]
encounters: 1
lastSeen: 2026-09-05T20:02:02Z
---

# `asset.import` over an existing asset faults inside `ForceReplaceReferences` and takes the editor down

## Symptom

A re-import of an existing texture killed a shared editor outright at
`2026-09-05T20:02:02Z`, six seconds after the Interchange import itself had already reported
success. Every agent in that editor lost its session.

```
[20.01.56:223][779] LogPinWrightSafePoint: Deferring work by one safe-point core-ticker hop (asset.import).
[20.01.56:243][779] LogInterchangeEngine: Display: Interchange start importing source
                    [X:/src/unreal/EAContentExamples58/Docs/fps/data/textures/T_WPN_Markings_M.png]
[20.01.56:247][779] LogTexture: Display: Building texture TwoD:
                    /Game/FPS/Weapons/Textures/T_WPN_Markings_M (TFO_AutoDXT, 2048x512)
[20.01.56:258][779] LogInterchangeEngine: Display: Interchange import completed [...]
[20.01.56:368][779] LogDatasmithContent: Do not use the UDatasmithStaticMeshCADImportData ...
[20.02.02:993][779] LogWindows: Error: === Critical error: ===
[20.02.02:993][779] LogWindows: Error: Unhandled Exception: EXCEPTION_ACCESS_VIOLATION
                    reading address 0xffffffffffffffff
```

Callstack, PinWright frames in bold order — the fault is reached from PinWright's own overwrite
path, not from an engine-initiated import:

```
FPropertyProxyArchive::operator<<()            PropertyProxyArchive.h:46
UStruct::SerializeExpr()                       ScriptSerialization.inl:243
UStruct::SerializeExpr()                       Class.cpp:2691
UStruct::Serialize()                           Class.cpp:2458
UFunction::Serialize()                         Class.cpp:7608
FFindReferencersArchive::ResetPotentialReferencer()  FindReferencersArchive.cpp:89
ObjectTools::ForceReplaceReferences'::<lambda_1>::operator()()  ObjectTools.cpp:1314
ObjectTools::ForceReplaceReferences()          ObjectTools.cpp:1369
ObjectTools::ForceReplaceReferences()          ObjectTools.cpp:1497
ObjectTools::ForceReplaceReferences()          ObjectTools.cpp:1504
AssetImportPolicy::FScopedOverwriteStage::Commit()   AssetImportPolicy.cpp:1010
AutoHandler_314_'::<lambda_1>::operator()()    AssetManageHandler.cpp:502
PinWrightSafePoint::DeferRequestToSafePoint    SafePoint.h:474
FRpcDispatcher::DeferActiveRequestToSafePoint  RpcDispatcher.cpp:1078
FEngineLoop::Tick()
```

Crash report: `Saved/Crashes/UECC-Windows-01FD4B9D466127197DF55DB7729358DA_0002`
(23:02:01 local = 20:02:01 UTC). Two earlier reports exist under the same session id
(`_0000` 21:52:19, `_0001` 22:16:33 local) — this editor had already died twice today.

## Mechanism

`ForceReplaceReferences` takes its `FThreadSafeObjectIterator` branch and calls
`FFindReferencersArchive::ResetPotentialReferencer` on **every live UObject**, which for a
`UFunction` serialises its compiled bytecode through `FPropertyProxyArchive`. `0xffffffffffffffff`
is not a null deref — it is a poisoned or stale `FProperty*` inside a script expression, so the
walk is reading a `UFunction` whose bytecode holds a dangling property pointer. Any Blueprint
recompiled earlier in the session can leave one behind: this editor had run
`blueprint.compile` on `/Game/FPS/Weapons/BP_Weapon_AR` at `20:01:45` and two
`blueprint.set_default` calls at `20:01:06` / `20:01:10`, eleven seconds before the fault.

**The blast radius is the whole process, and it is not the caller's asset.** The import had
already succeeded; what died was the reference fix-up afterwards. Nothing the importing caller
passed can make this safe, and nothing any *other* agent in the editor did can protect it —
three other streams were mid-work in this editor and all lost their sessions.

`B-asset-delete-force-delete-leaves-uasset-on-disk` / `B-force-delete-nulls-referencers` already
document `ObjectTools::ForceReplaceReferences` as an unsafe hammer on the **delete** path.
This ticket is the **import/overwrite** path reaching the same function, and it is worse there
because the delete path at least intends to destroy something: an import that overwrites a
texture has no reason to rewrite references at all — the `UTexture2D` object identity is
preserved by a re-import, so nothing needs replacing.

## Expected

1. **Do not call `ForceReplaceReferences` on a same-class re-import.** When the overwrite target
   and the newly imported object are the same `UObject` (or the same class at the same path),
   the reference set is unchanged; the commit stage should be a no-op. That alone removes this
   crash for the ordinary "re-import the texture I just regenerated" case, which is what every
   producer-driven workflow in this project does.
2. When a replace genuinely is needed (class changed, or the import produced a different object),
   use the same guarded path the delete tickets ask for, and **wrap the walk** so a poisoned
   referencer is reported rather than fatal.
3. Failing both, refuse with a typed error before importing: an `asset.import` that would run
   `ForceReplaceReferences` in an editor that has compiled a Blueprint this session is a coin
   flip on the whole process, and a refusal costs one caller a retry instead of costing every
   caller their session.

## Repro (shape, not yet minimised)

1. Shared editor, several agents working.
2. `blueprint.compile` and/or `blueprint.set_default` on any Blueprint.
3. `asset.import` a PNG over an **existing** texture asset path
   (`Docs/fps/data/textures/T_WPN_Markings_M.png` → `/Game/FPS/Weapons/Textures/T_WPN_Markings_M`).
4. Import reports success; ~6 s later the editor faults in `ForceReplaceReferences`.

Not minimised because reproducing it costs an editor. The log window above is complete and the
callstack is unambiguous; the ordering with the Blueprint compiles is the part that needs a
controlled run to confirm as *necessary* rather than merely present.

## Reporter's position

**I was not the caller.** I was authoring `/Game/FPS/Weapons/Materials/M_WPN_OpticLens` in the
same editor and the crash ended my session; the import belonged to the markings stream. Filed
because an editor-killer that costs three other streams their work should not wait on the one
agent who happened to issue the call, and because the callstack lands squarely in
`AssetImportPolicy.cpp`. If the importing stream files its own, merge these.

## History
- `#1-forcereplacereferences-av-on-texture-reimport` `OPEN` reporter — First encounter, `2026-09-05T20:02:02Z`, UE 5.8. `asset.import` of `T_WPN_Markings_M.png` over the existing `/Game/FPS/Weapons/Textures/T_WPN_Markings_M`; Interchange reported the import complete at `20:01:56`, then `FScopedOverwriteStage::Commit` → `ObjectTools::ForceReplaceReferences` → `FFindReferencersArchive::ResetPotentialReferencer` → `UFunction::Serialize` → `FPropertyProxyArchive::operator<<` faulted with `EXCEPTION_ACCESS_VIOLATION reading 0xffffffffffffffff`. Crash report `UECC-Windows-01FD4B9D466127197DF55DB7729358DA_0002`; the same session id had already produced `_0000` and `_0001`. Distinct from `B-asset-import-unconditional-replace-hidden-outputs` (that ticket is about the overwrite being unconditional and multi-output results being hidden — it does not name a crash) and from the two `asset.delete` `ForceReplaceReferences` tickets (different verb, different stage). Ask: skip the replace entirely on a same-class re-import, where object identity is preserved and nothing needs replacing.
