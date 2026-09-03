---
id: B-generate-thumbnail-package-path-transient-not-found
title: "EditorAssetLibrary existence and listing helpers intermittently answer \"does not exist\" for assets that do, in a shared editor under concurrent writes — asset.generate_thumbnail surfaces it as ASSET_NOT_FOUND, and a false negative here invites a caller to re-create and clobber a live asset"
status: OPEN
severity: High
category: bug
tags: [thumbnail, asset-path, asset-registry, shared-editor, false-negative, editor-asset-library]
encounters: 2
lastSeen: 2026-09-03T07:12:00+05:00
---

# `asset.generate_thumbnail` refuses a package path it accepted minutes earlier

`asset.generate_thumbnail {assetPath: "/Game/FPS/Env/Meshes/SM_ENV_Truck"}` returned

```
[ASSET_NOT_FOUND] Asset not found
```

twice in a row, for an asset that

- had been rendered by **the same call with the same argument** eight minutes earlier,
- was on disk (`Content/FPS/Env/Meshes/SM_ENV_Truck.uasset`, 535734 bytes, mtime 04:31:32), and
- resolved in the same shared editor, seconds later, from `python.execute`:
  `EditorAssetLibrary.does_asset_exist` -> `True`,
  `EditorAssetLibrary.load_asset` -> `<Object '/Game/FPS/Env/Meshes/SM_ENV_Truck.SM_ENV_Truck' Class 'StaticMesh'>`,
  `AssetRegistry.get_asset_by_object_path` -> valid.

Passing the **object** path instead — `/Game/FPS/Env/Meshes/SM_ENV_Truck.SM_ENV_Truck`, same
call, same options — succeeded on the first try and wrote a correct 768x768 PNG. A later
package-path call on the same asset then also succeeded.

So the verb has two resolution outcomes for one asset and the failing one is not sticky.

## What was expected

The verb documents `assetPath` as "Path to the asset" and every other PinWright verb in the
session took the package form. Either form should resolve, and a verb that cannot resolve an
asset that `does_asset_exist` reports `True` for should say what it actually failed to do.

## Not reproduced on demand

This is filed as an observation with exact readings, not a deterministic repro. Two attempts to
force the state afterwards failed: `SystemLibrary.collect_garbage()` did not unload the mesh
(still referenced), and package-path calls on the same and on a sibling asset succeeded
afterwards. The suspicion is that the package-path branch resolves only an **already-resident**
object while the object-path branch loads, which would make the failure a function of what the
shared editor happens to be holding; that is a guess and the source was not read.

Context that may matter: this is a shared editor with six other agent streams live, and the
failing window was immediately after a batch of `model.compile` and `asset.nanite_rebuild_mesh`
writes on neighbouring assets, so async asset compilation was in flight.

## Workaround

Pass the object path (`/Package/Name.Name`) whenever a package path returns `ASSET_NOT_FOUND`
for an asset that exists; it resolved on the first attempt every time.

## History
- `#1-filed` `OPEN` reporter — Observed 2026-09-03, UE 5.8, PinWright at this checkout's HEAD, building the FPS compound map (map as forcing function; host `CLAUDE.md` § "What this project is for"). `asset.generate_thumbnail` with `assetPath: "/Game/FPS/Env/Meshes/SM_ENV_Truck"` returned `[ASSET_NOT_FOUND] Asset not found` twice while the same asset resolved through `does_asset_exist`, `load_asset` and the asset registry in the same second, and while the identical call on `/Game/FPS/Env/Meshes/SM_ENV_Barrier_Jersey` in the same batch succeeded. The object-path form `/Game/FPS/Env/Meshes/SM_ENV_Truck.SM_ENV_Truck` succeeded immediately; a package-path retry a minute later also succeeded, so the state cleared on its own. Not reproducible on demand — `collect_garbage()` did not unload the mesh and later package-path calls all succeeded — so this is recorded as evidence rather than as a repro, with the working hypothesis that the package-path branch resolves only already-resident objects while the object-path branch loads. Filed rather than dropped because a false `ASSET_NOT_FOUND` is indistinguishable, to a caller, from an asset that was never written, which is exactly the confusion this build was in the middle of (three prior tickets on this project turned on writes that reported success and did not land).
- `#2-underlying-existence-check-is-the-bug` `OPEN` reporter — **Root cause narrowed, severity raised Medium -> High, title rewritten.** `#1` framed this as a thumbnail path-resolution quirk. It is not: the existence check underneath it is unreliable, and `asset.generate_thumbnail` is only where it happened to surface. Same session, same shared editor (six other agent streams compiling and saving concurrently), UE 5.8, PinWright at this checkout's HEAD.

  **Readings, in order, all from `python.execute` in the one editor.**

  02:09 — two assets present on disk (`Content/FPS/Env/Cine/MPC_ENV_Hero_4K.uasset`, `LS_ENV_Hero.uasset`) and correctly indexed by the asset registry:
  ```
  AssetRegistry.get_assets_by_path('/Game/FPS/Env/Cine', recursive=True) -> 2 rows
      /Game/FPS/Env/Cine/MPC_ENV_Hero_4K   MoviePipelinePrimaryConfig
      /Game/FPS/Env/Cine/LS_ENV_Hero       LevelSequence
  AssetRegistry.is_loading_assets()                                      -> False
  AssetRegistry.get_sub_paths('/Game/FPS/Env')  -> [Cine, Materials, Meshes, Textures]
  ```
  while every `EditorAssetLibrary` answer for the same paths was negative:
  ```
  EditorAssetLibrary.does_asset_exist('/Game/FPS/Env/Cine/MPC_ENV_Hero_4K')  -> False
  EditorAssetLibrary.load_asset(... package path or object path ...)         -> None
  EditorAssetLibrary.does_directory_exist('/Game/FPS/Env/Cine')              -> False
  EditorAssetLibrary.does_directory_exist('/Game/FPS/Env/Meshes')            -> False
  EditorAssetLibrary.does_directory_exist('/Game/FPS/Env/Materials')         -> False
  EditorAssetLibrary.list_assets('/Game/FPS/Env/Cine', recursive=True)       -> []
  ```
  The registry and `EditorAssetLibrary` disagreed about the same four directories at the same instant.

  **The negative was not confined to those two assets.** In the same call, `does_asset_exist('/Game/FPS/Env/Meshes/SM_ENV_Truck')` returned `False` — an asset this session had compiled, saved, Nanite-enabled and thumbnailed minutes earlier and which was resident in memory.

  **It cleared by itself inside a minute.** The very next call, no intervening write, probed five paths across three mount roots and every one answered `True` with a live object: `/Game/FPS/Env/Meshes/SM_ENV_Truck`, `/Game/FPS/Weapons/Props/SM_Prop_Crate`, `/Game/FPS/Maps/FPS_Compound`, `/Engine/EngineMaterials/DefaultMaterial`, `/Game/Dota2/Materials/MI_tree_dire_dark`; and `does_directory_exist` was `True` for `/Game`, `/Game/FPS`, `/Game/FPS/Env`, `/Engine` and `/Game/Dota2`. Three repeats immediately after were all `True`.

  **Escape hatch that worked while the negatives were live:** `unreal.load_package('/Game/FPS/Env/Cine/MPC_ENV_Hero_4K')` returned the `Package`, and `unreal.find_asset('/Game/FPS/Env/Cine/MPC_ENV_Hero_4K.MPC_ENV_Hero_4K')` then returned the `MoviePipelinePrimaryConfig`. So the data was reachable throughout; only the `EditorAssetLibrary` path was answering wrongly.

  **Why this is High, not Medium.** A false "does not exist" is not a read failure, it is an invitation to write. The standard create-if-absent shape — `if not does_asset_exist(p): create(p)` — is used across PinWright's own create verbs and by every agent script on this project; under this bug it silently re-creates and clobbers a live asset. That is the same failure family as the tickets this project keeps paying for, except the loss is the caller's data rather than a wasted call. It also makes any PinWright `ASSET_NOT_FOUND` untrustworthy as evidence that an asset is missing.

  **Not read in source.** No `file:line` claim is made here: this was measured through Python, not traced through `UEditorAssetLibrary`. The narrowing worth having for whoever does read it is that the registry's own query path (`get_assets_by_path`, `get_sub_paths`, `is_loading_assets`) was correct at the same instant the `EditorAssetLibrary` wrappers were wrong, so the fault is in the wrapper's lookup, not in the registry contents; and `is_loading_assets() == False` rules out the obvious "still scanning" explanation.

  **Wanted:** PinWright verbs should not resolve an asset through a check that can answer wrongly. Resolve through the registry (which was right), or load and then test the object, and never report `ASSET_NOT_FOUND` without having tried a load. **Workaround now in force on this build:** treat a lone `ASSET_NOT_FOUND` as unproven — retry once, and confirm with `unreal.load_package` + `unreal.find_asset` before concluding anything is missing. `encounters` 1 -> 2, `lastSeen` refreshed.

- `#3-deterministic-repro-one-call` `OPEN` reporter — **No longer intermittent: there is a one-call repro.** `#1` and `#2` recorded this as a window that clears by itself. On `/Game/FPS/Env/Cine/MPC_ENV_Hero_4K` it does not clear — it reproduces on every attempt, including inside the same `python.execute` that then resolves the object:
  ```
  EditorAssetLibrary.does_asset_exist('/Game/FPS/Env/Cine/MPC_ENV_Hero_4K')  -> False
  EditorAssetLibrary.load_asset      ('/Game/FPS/Env/Cine/MPC_ENV_Hero_4K')  -> None
  unreal.load_package                ('/Game/FPS/Env/Cine/MPC_ENV_Hero_4K')  -> <Package ...>
  unreal.find_asset('/Game/FPS/Env/Cine/MPC_ENV_Hero_4K.MPC_ENV_Hero_4K')
      -> <Object '.../MPC_ENV_Hero_4K.MPC_ENV_Hero_4K' Class 'MoviePipelinePrimaryConfig'>
  ```
  Three lines apart, one path, two answers. `LS_ENV_Hero` in the same folder behaves identically. Repeated across four separate calls several minutes apart, with and without the package already resident, so it is not a warm/cold distinction either — after `load_package` had made the object resident, the next call's `does_asset_exist` was still `False`.

  So `#2`'s "clears by itself" describes the *broad* form (unrelated paths going negative together under concurrent load) but understates it: at least some assets are permanently invisible to `EditorAssetLibrary` while being fully present, registry-indexed and loadable by other routes. Both assets were created earlier in this same project by PinWright verbs, live under `/Game/FPS/Env/Cine/`, and are byte-present on disk.

  This makes the ticket actionable without waiting for a race to recur: `MPC_ENV_Hero_4K` is a standing fixture in this checkout. `encounters` unchanged (same session), narrowing only.

- `#4-object-path-workaround-retracted-and-residency-ruled-out` `OPEN` reporter — **Correcting `#1` and `#3`: the object-path form is NOT a reliable workaround, and "the asset is not resident" is NOT the mechanism.** Both were stated in this ticket and both are wrong; recorded here rather than edited away.

  Reproduced on a third asset, `/Game/FPS/Env/Meshes/SM_ENV_RoofBallast`, ~35 minutes after `#2`/`#3`, same editor:
  ```
  asset.generate_thumbnail {assetPath: ".../SM_ENV_RoofBallast.SM_ENV_RoofBallast"}   [ASSET_NOT_FOUND]   (object path)
  asset.generate_thumbnail {assetPath: ".../SM_ENV_RoofBallast.SM_ENV_RoofBallast"}   [ASSET_NOT_FOUND]   (retried, identical)
  ```
  `#1` reported the object-path form succeeding where the package path failed; on this asset **both forms are refused**, so that escape hatch does not generalise and should not be relied on.

  In the same second, from `python.execute`:
  ```
  EditorAssetLibrary.does_asset_exist('/Game/FPS/Env/Meshes/SM_ENV_RoofBallast')   -> False
  EditorAssetLibrary.load_asset      ('/Game/FPS/Env/Meshes/SM_ENV_RoofBallast')   -> None
  unreal.find_asset('/Game/FPS/Env/Meshes/SM_ENV_RoofBallast.SM_ENV_RoofBallast')
      -> <Object '.../SM_ENV_RoofBallast.SM_ENV_RoofBallast' (0x0000024518B51000) Class 'StaticMesh'>
  EditorAssetLibrary.does_directory_exist('/Game/FPS/Env/Meshes')                  -> False
  ```
  **`find_asset` returning a live pointer proves the object is resident in memory.** `find_asset` does not load; it only looks up an already-constructed UObject. So `#2`'s working hypothesis — that the failing branch resolves only already-resident objects while the succeeding branch loads — is refuted from the other direction: the object IS resident and `load_asset` still returns `None` for it. Whatever `EditorAssetLibrary`'s lookup consults, it is not the object table.

  Third distinct asset (`MPC_ENV_Hero_4K`, `LS_ENV_Hero`, now `SM_ENV_RoofBallast`), third distinct directory reported non-existent (`/Game/FPS/Env/Cine`, `/Game/FPS/Env/Materials`, `/Game/FPS/Env/Meshes`) while the registry lists all of them.

  **Only reliable route left, and the one now in force on this build:** `unreal.find_asset('<package>.<name>')`, after `unreal.load_package('<package>')` if it returns None. Any PinWright verb that resolves a caller's asset path should do the same; none of `does_asset_exist`, `load_asset`, `does_directory_exist` or `list_assets` can be trusted as a gate.
