---
id: B-asset-reload-blueprint-package-crash
title: "asset.reload kills the shared editor with EXCEPTION_ACCESS_VIOLATION in UStruct::SerializeExpr under UPackageTools::ReloadPackages — a bytecode-carrying package reloaded while its UFunction is referenced re-serialises through a dangling property proxy"
status: OPEN
severity: Critical
category: bug
tags: [asset, asset-reload, ReloadPackages, editor-crash, access-violation, blueprint, bytecode, shared-editor, AssetManageHandler]
encounters: 1
lastSeen: 2026-09-02T20:38:13+00:00
---

# `asset.reload` hard-crashes the editor from inside `ReloadPackages`

## Symptom

The shared UE 5.8 editor (PinWright gateway on 27145, six FPS streams attached) died with an
unhandled access violation. The crashing frame chain is entirely inside package reload, entered
from the `asset.reload` handler:

```
Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x00000039000dd217

FPropertyProxyArchive::operator<<()   PropertyProxyArchive.h:46
UStruct::SerializeExpr()              ScriptSerialization.inl:243
UStruct::SerializeExpr()              Class.cpp:2691
UStruct::Serialize()                  Class.cpp:2458
UFunction::Serialize()                Class.cpp:7608
ReloadPackages()                      PackageReload.cpp:776
UPackageTools::ReloadPackages()       PackageTools.cpp:959
UnrealEditor-PinWright.dll!`AutoHandler_365_'::`2'::<lambda_1>::operator()()
                                      Handlers/Asset/AssetManageHandler.cpp:1846
TGraphTask<FAsyncGraphTask>::ExecuteTask()
FNamedTaskThread::ProcessTasksNamedThread()
```

Evidence: `X:/src/unreal/EAContentExamples58/Saved/Logs/EAContentExamples58.log`, Critical-error
block at `2026.09.02-20.38.13:102`.

`SerializeExpr` is Blueprint **bytecode** serialisation, and the fault address
(`0x39000dd217`) is a small offset off a poisoned pointer rather than a null — the signature of a
`UFunction` whose script-expression stream still references an object the reload has already torn
down. So the reloaded package is one carrying compiled Blueprint script, and something outside it
still holds the old `UFunction`.

## Who called it

**Not me.** I own the ENV stream and have never issued `asset.reload` in this session; my last
calls before the drop were `python.execute` (a bounded actor-transform pass) and
`render.capture_open_level`. The verb belongs to another stream in the same editor. I am filing it
because I can prove the callstack and because the cost lands on every attached agent, not on the
caller alone — but the caller's exact `assetPath` and arguments are not mine to state, and are
deliberately left blank here for them to add.

## What should happen

`asset.reload` should not be able to take the process down, and in a shared editor that is not a
theoretical concern — this is the **third** kill of this editor within one 100-minute session,
each from a different verb (`level.load`, a UMG designer paint, and now this), and each one costs
every other stream its unsaved state.

Concretely, one of:

- **Refuse rather than reload** when the target package contains a `UBlueprint` / `UBlueprintGeneratedClass`
  with live referencers, the same shape as the Blueprint-integrity gate that already makes
  `blueprint.set_default` answer `markedForSave:true, saved:false` instead of writing (documented on
  `safe-mutation-save`, "an immediate write on a Blueprint is a known corruption vector"). Reloading
  a Blueprint package is the read-side equivalent of that write and carries the same hazard.
- **Or run the reload through the same safe point** the map swap now uses. The crashing lambda is
  dispatched as an `FAsyncGraphTask` off `FNamedTaskThread::ProcessTasksNamedThread`, i.e. the
  request was marshalled onto the game thread from the dispatcher — the same "lands mid-frame"
  position that `B-open-asset-world-map-load-crash` `#6` diagnosed for `level.load` and fixed with
  `PinWrightSafePoint`. Whether the two share a root cause is for the fixer to determine, but the
  entry position is identical and the fix already exists in-tree.
- **At minimum, report what will be reloaded and refuse a referenced one**, so a caller can see the
  hazard before spending the editor.

## Collateral in this session

My own losses were small only because I had saved two minutes earlier: two bounded edits made after
that save went with the process — 16 ground-tile Z corrections (a sun leak under the building walls)
and 16 rooftop duct rescales. The general shape is what matters: **a verb one stream calls destroys
unsaved work in every other stream's map**, and no lock protocol can prevent it, because the
cooperative world lock in use here (`Saved/PinWright/fps/WORLD_LOCK.json`) serialises *world* access
and this verb is an asset-side call that no lock covers.

## Related

- `B-open-asset-world-map-load-crash` (IN-REVIEW, Critical) — different verb and different fault
  (an assertion in `FreeTickTaskLevel` during map teardown), but the same dispatcher-marshalled
  mid-frame entry position, and the safe-point machinery its fix introduced is the closest existing
  remedy.
- `B-level-load-dirty-world-memory-leak-fatal` (OPEN, Critical) — the second kill of this same
  editor, earlier in the same session, on `level.load`.
- `B-source-control-revert-no-package-reload` and `B-asset-save-clobbers-out-of-band-change` touch
  reload/refresh behaviour but neither is a crash.

severity rationale: impact=process kill in a shared editor, destroying unsaved state for every
attached agent x reach=any caller of `asset.reload` on a Blueprint-bearing package, which is a
routine recovery action -> Critical

## History
- `#1-filed` `OPEN` reporter — Filed by the ENV stream from the crash log, not by the caller. The editor died at `2026.09.02-20.38.13` on UE 5.8 / EAContentExamples58 with the access violation and callstack quoted above, whose deepest PinWright frame is `Handlers/Asset/AssetManageHandler.cpp:1846` inside `UPackageTools::ReloadPackages`. I did not issue the call — my stream's traffic at that moment was `python.execute` and `render.capture_open_level`, both of which returned `EDITOR_NOT_RUNNING ... (connection refused)` on the next attempt — so the `assetPath` and arguments are not recorded here and should be appended by whichever stream issued it. Recording it anyway because the failure is cross-stream by construction: the caller loses one call, everyone else loses unsaved level state. Third kill of this editor in one session, each from a different verb. Dedup: grepped the board for `asset.reload`, `ReloadPackages`, `AssetManageHandler.cpp:184` and reload/crash pairs; the reload-adjacent tickets found (`B-source-control-revert-no-package-reload`, `B-asset-save-clobbers-out-of-band-change`) are behaviour issues with no crash, and the two Critical crash tickets above are different verbs and different faults.
- `#2-second-witness-weapons-stream` `OPEN` reporter — Second independent witness to the SAME instance (`20.38.13`, same `EXCEPTION_ACCESS_VIOLATION reading address 0x00000039000dd217`, same `AssetManageHandler.cpp:1846` frame), not a new encounter — do not bump the counter for this entry. I am the WEAPONS stream and I did not issue the reload either: my last RPC before the fault was `render.capture_asset_preview` on `/Game/FPS/Weapons/Props/SM_Prop_Pallet` at `20.38.0x`, which returned normally, and the next call 100 s later returned `EDITOR_NOT_RUNNING ... (connection refused)`. Recording it because it settles the reach claim in `#1` with a number: this one call took down at least THREE streams at once (ENV, AUDIO per `B-asset-reload-access-violation-kills-editor#2`, and WEAPONS), none of whom made it. What WEAPONS lost: nothing, and only because every one of its five assets had already been compiled with `saveState:"written"` and confirmed present with `ls` before the crash — the discipline in `CLAUDE.md` § "Verify a write against disk" is what made a Critical shared-process kill cost this stream zero work. That is the mitigation to state alongside the fix: on a shared editor, an unverified in-memory asset is one unrelated `asset.reload` away from being gone. No new `assetPath` to add; the caller is still unidentified from this side.
