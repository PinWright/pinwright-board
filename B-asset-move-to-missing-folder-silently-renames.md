---
id: B-asset-move-to-missing-folder-silently-renames
title: "asset.move given a destination folder that does not exist renames the asset to the folder's last path segment instead of moving it, and reports success:true with the requested path echoed back — the handler documents folder destinations and implements none"
status: OPEN
severity: High
category: bug
tags: [asset, asset-move, asset-rename, destinationPath, docs-behaviour-mismatch, silent-wrong-data, echoed-not-measured, false-success, housekeeping]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Asked to move into a folder, it renamed the asset after the folder

Measured live:

```js
call({method: "asset.move", args: {
  sourcePath: "/Game/Landscape/LGT_VegTest_MeadowBase",
  destinationPath: "/Game/VegetationTest/Grass" }})
// -> success: true, assetName: "Grass"
```

`/Game/VegetationTest` did not exist. The asset is now called **`Grass`**, it is in the old folder,
and the name it had is gone. The response said the move succeeded.

## The handler documents folder destinations and implements none

The parameter doc is explicit
(`Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp:532`):

> `destinationPath` — "Target folder or full asset path (e.g. /Game/New/SM_Foo). **When a folder is
> given, the asset retains its original name.**"

There is no code behind the second sentence. The body's only path-shaping step is
`AssetManageHandler.cpp:545-552`:

```cpp
if (!DestinationPath.IsEmpty() && FPaths::GetPath(DestinationPath).IsEmpty())
{
    FString ParentDir = FPaths::GetPath(SourcePath);
    ...
    DestinationPath = ParentDir / DestinationPath;
}
```

That branch fires only for a **bare name with no slash in it**. Any `destinationPath` containing a
slash — which every folder path does — skips it and goes verbatim into
`UEditorAssetLibrary::RenameAsset(ResolvedSourcePath, DestinationPath)` at `:567`. Nothing anywhere
in the handler asks whether `destinationPath` names an existing folder; `DoesAssetExist` is called
on the *source* only (`:560`).

`RenameAsset`'s second parameter is a **full destination object path**, not a folder. The engine
splits it (`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/Subsystems/EditorAssetSubsystem.cpp:1047-1052`):

```cpp
FString DestinationLongPackagePath = FPackageName::GetLongPackagePath(DestinationObjectPath);
FString DestinationObjectName      = FPackageName::ObjectPathToObjectName(DestinationObjectPath);
AssetToRename.Add(FAssetRenameData(SourceObject, DestinationLongPackagePath, DestinationObjectName));
```

(`UEditorAssetLibrary::RenameAsset` -> `UEditorAssetSubsystem::RenameAsset`
(`EditorAssetLibrary.cpp:387-391`) -> `UE::EditorAssetUtils::RenameLoadedAsset`
(`EditorAssetSubsystem.cpp:1018-1056`).)

So `/Game/VegetationTest/Grass` is *always* read as "package path `/Game/VegetationTest`, object
name `Grass`". The last segment becomes the asset's new name, unconditionally. The documented
folder case is unreachable for any folder whose leaf name is not already the asset's name.

## The success is an echo, not a measurement

`:571` publishes `assetPath` straight from the request string:

```cpp
Resp->SetStringField(TEXT("assetPath"), DestinationPath);
```

and the only measured field comes from `AddAssetVerification(Resp, MovedAsset)` at `:576`, which is
guarded by `UObject* MovedAsset = UEditorAssetLibrary::LoadAsset(DestinationPath);` at `:573` and
**skipped in silence** when that load returns null (`AddAssetVerification` itself early-returns on a
null asset — `Utils/AssetUtils.cpp:1753-1758`). So the response can assert the requested destination
whether or not the asset arrived there, and its own verification block is the thing that goes
missing when it did not.

## What is attributed and what is not

Attributed, from the source above: the last path segment is taken as the new object name, the
handler has no folder branch, the destination is never probed, and `assetPath` is an echo. That is
sufficient to explain "renamed instead of moved" and "reported success".

**Not attributed:** why the asset ended up in the *old* folder rather than in a newly created
`/Game/VegetationTest`. The engine path above hands `/Game/VegetationTest` to `IAssetTools::RenameAssets`
as the destination package path, and what that does with a package path that has no folder on disk
was not traced. The observation is the zone agent's measurement — asset named `Grass`, still under
`/Game/Landscape` — and it is recorded as an observation, not explained here. Whoever fixes this
should establish that half before assuming the rename is confined to the name.

## Impact

This is a general asset-verb defect that will bite anyone; it only appears in a vegetation report by
accident. `asset.move` is ordinary housekeeping — the natural response to `landscape.create_grass_type`'s
hardcoded `/Game/Landscape` output path (see `E-create-grass-type-cull-distance-and-output-path`) is
to move the asset somewhere sensible, and that is exactly the call that destroys its name.

Three consequences in the order a caller meets them:

1. **The asset's name is gone** and nothing announced it. The name is how every other verb, every
   note and every script refers to it.
2. **The reported `assetPath` is wrong**, so the next call in the chain is aimed at a path that does
   not hold what the caller thinks.
3. **A caller following the documentation gets this**, because the doc explicitly invites a folder
   destination. This is not a misuse.

**Workaround:** always pass a full destination *object* path — `/Game/New/SM_Foo`, with the leaf
equal to the asset name you want — and never a bare folder. Create the destination folder first if
it does not exist. Verify afterwards from something other than the response's `assetPath`.

## Fix

1. **Detect the folder case and implement the documented behaviour.** When `destinationPath`
   resolves to an existing content folder, append the source's leaf name and move there. This is
   what `:532` already promises.
2. **Refuse the ambiguous case rather than guessing.** When `destinationPath` names neither an
   existing folder nor a path whose parent folder exists, return a typed error naming both readings
   ("did you mean to move into the folder X, or to rename to X?") instead of silently picking the
   rename. A rename the caller did not ask for is the worse of the two guesses because it is the
   one that discards information.
3. **Measure the result.** `assetPath` must come from the moved object, not from the request. The
   null-`MovedAsset` branch at `:573-577` must be an error, not a silent omission — if the asset is
   not loadable at the destination, the move did not do what the response says.

## Same shape as

- `B-bulk-rename-docs-behaviour-mismatch` (IN-REVIEW, High) — a mutating asset verb whose behaviour
  contradicts its own parameter documentation, producing wrong asset names that need a second
  destructive pass to undo. Same class, sibling verb, different mismatch: that one is
  case-sensitivity and operation order inside `asset.bulk_rename`, this is folder-vs-object-path in
  `asset.move`. Not a duplicate — different file, different function, no shared fix.
- `B-asset-verification-clobbers-handler-assetpath` (IN-REVIEW, High) — `assetPath` in a response
  not being the value the verb set. Adjacent but distinct: that is the verification block
  *overwriting* a correct handler value; this is the handler value never being measured at all, and
  the verification block being skipped.
- `B-asset-delete-force-delete-leaves-uasset-on-disk` (IN-REVIEW, High) — an asset verb whose
  success report and the state on disk disagree. Same family of "the write verb's answer is not
  evidence".

## Severity

**High**, by impact class: *silent wrong data on a normal path*, plus a silent false-success. The
caller asked for a move, got a rename, and was told it worked.

**Critical considered and declined, deliberately.** The Critical band is "a write that corrupts or
loses asset data". The argument for it: an asset silently renamed, with the old name gone and the
response asserting a path the asset is not at, is arguably lost data — a caller who does not notice
has a broken reference in every note and script that named it.

Declined on three grounds, all checkable:

- **The asset's data is untouched.** The bytes, the object, and its properties are intact; only its
  name and package changed.
- **A redirector is left at the old path.** That is the verb's own contract ("Move an asset to a
  different folder and leave a redirector at the old path", `:529`), and it is what
  `IAssetTools::RenameAssets` does, so existing hard references still resolve. Nothing dangles until
  someone runs `asset.fixup_redirectors`.
- **It is reversible if noticed** — a second `asset.move` with the correct full object path puts it
  back, at the cost of another redirector.

So the damage is a wrong name plus a false report, not destruction. That is squarely the High band's
"the caller trusts a result that is a lie and builds on it".

**Reach modifier declined.** `asset.move` is common housekeeping but not an every-session verb, so
no bump in either direction; High stands on impact alone.

## History
- `#1-move-to-missing-folder-renames` `OPEN` reporter — Measured live against a running editor:
  `asset.move {sourcePath: "/Game/Landscape/LGT_VegTest_MeadowBase", destinationPath: "/Game/VegetationTest/Grass"}`
  with `/Game/VegetationTest` non-existent returned `success: true, assetName: "Grass"`; the asset is
  now named `Grass`, in the old folder, and its former name is gone. Mechanism re-derived:
  `AssetManageHandler.cpp:532` documents a folder destination, `:545-552` is the only path-shaping
  branch and fires only for a slash-free bare name, `:567` passes the string verbatim to
  `UEditorAssetLibrary::RenameAsset` whose destination is a full object path — split into package
  path + object name at `EditorAssetSubsystem.cpp:1047-1052` via
  `EditorAssetLibrary.cpp:387-391` and `EditorAssetSubsystem.cpp:1018-1056`. The response's
  `assetPath` is an echo of the request (`:571`) and the measured verification block is skipped in
  silence when the destination load returns null (`:573-577`, `AssetUtils.cpp:1753-1758`).
  **Not attributed:** why the asset stayed in the old folder rather than landing in a created
  `/Game/VegetationTest` — recorded as an observation, not explained. Severity argued Critical vs
  High in the body; High chosen because the asset data is intact, a redirector is left at the old
  path per the verb's own contract, and the rename is reversible.
