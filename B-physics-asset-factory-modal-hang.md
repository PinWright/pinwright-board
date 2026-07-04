---
id: B-physics-asset-factory-modal-hang
title: "physics.setup_physics_simulation & skeleton.create_physics_asset (mesh-backed) hang the editor — UPhysicsAssetFactory opens a modal body-generation dialog on the game thread that an unattended MCP session can never answer"
status: OPEN
severity: High
category: bug
tags: [physics-asset-factory-modal-hang, game-thread-hang, modal-dialog, physics, skeleton, ragdoll, setup_physics_simulation, create_physics_asset, unattended-editor]
encounters: 1
lastSeen: 2026-07-05T00:14:45.8025931+03:00
---

# `physics.setup_physics_simulation` (and mesh-backed `skeleton.create_physics_asset`) wedge the editor via `UPhysicsAssetFactory`'s modal body-generation dialog

## What's wrong

Calling `physics.setup_physics_simulation` against a real `USkeletalMesh` hangs
the editor game thread indefinitely. No `UPhysicsAsset` is created, no error is
returned, and every subsequent RPC — including a trivial read-only
`actor.list {}` — times out on the proxy's game-thread liveness ping. The editor
process stays alive and `Responding: True` (Slate keeps pumping a modal message
loop) with CPU spinning (~37% observed), but the RPC dispatcher, which marshals
onto the game thread, never runs again. Recovery requires OS-killing the editor.

The root cause is shared with `skeleton.create_physics_asset`'s **mesh-backed
branch**: both hand a real `USkeletalMesh` to `UPhysicsAssetFactory` and invoke
it on the game thread, and the engine factory opens the interactive
"New Physics Asset" body-generation options modal (`SPhysicsAssetGenerationDialog`
via `GEditor->EditorAddModalWindow`) whenever `TargetSkeletalMesh` is set and the
app is not running unattended. This fuzz host runs a **full interactive editor**
(the MCP drives a normally-launched editor, not `-unattended`), so
`FApp::IsUnattended()` is false, the modal shows, and it blocks the game thread
forever waiting for a click that no one will make.

## Affected methods (family)

- `physics.setup_physics_simulation` — whenever it resolves a real `USkeletalMesh`
  (via `meshPath`, via `actorName` -> the actor's `SkeletalMeshComponent`, or via
  `skeletonPath` whose skeleton has a preview mesh). This is the verb that hung
  this iteration.
- `skeleton.create_physics_asset` — the **mesh-backed** branch only
  (`skeletalMeshPath` resolves to a `USkeletalMesh`), which calls
  `Factory->FactoryCreateNew(...)` directly with the same `TargetSkeletalMesh`.

NOT affected: `skeleton.create_physics_asset`'s **bare-skeleton** branch
(`PhysicsAssetHandler.cpp:284-313`), which builds capsule bodies manually via the
in-plugin `BuildBodiesFromSkeletonRefPose` helper and never touches the factory —
so a bare `USkeleton` payload (`fromSkeleton:true`) completes headlessly. The
positive replay in `B-create-physics-asset-skeletonpath-alias-dead` (step 3,
`bodyCount:1, fromSkeleton:true`) exercised exactly that safe branch, which is
why that path was never observed to hang; the mesh-backed factory path is
untested by that ticket.

## Root cause (ground truth from plugin source)

`physics.setup_physics_simulation` — `Plugins/PinWright/Source/PinWright/Private/Handlers/Physics/PhysicsHandler.cpp:421-426`:

```cpp
  PhysicsFactory->TargetSkeletalMesh = TargetMesh;

  FAssetToolsModule &AssetToolsModule =
      FModuleManager::LoadModuleChecked<FAssetToolsModule>("AssetTools");
  UObject *NewAsset = AssetToolsModule.Get().CreateAsset(
      PhysicsAssetName, SavePath, UPhysicsAsset::StaticClass(), PhysicsFactory);
```

`skeleton.create_physics_asset` (mesh-backed) — `Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/PhysicsAssetHandler.cpp:315-325`:

```cpp
    UPhysicsAssetFactory* Factory = NewObject<UPhysicsAssetFactory>();
    ...
    Factory->TargetSkeletalMesh = SkeletalMesh;

    UObject* NewAsset = Factory->FactoryCreateNew(UPhysicsAsset::StaticClass(), Package,
                                                   FName(*AssetName), RF_Public | RF_Standalone,
                                                   nullptr, GWarn);
```

Both reach the engine's `UPhysicsAssetFactory::FactoryCreateNew` (Editor/UnrealEd)
with `TargetSkeletalMesh` set. With a mesh present and the app not unattended,
that factory pops the modal body-setup options window on the game thread; the
modal loop pumps Slate (hence `Responding: True` and non-zero CPU) but never
returns control to the RPC dispatcher. There is no in-plugin log line between the
call arriving and the freeze because the handler emits no `UE_LOG` on the
`meshPath` path before the `CreateAsset` call.

## Repro (this iteration — the hang IS the repro)

Attempt drove `physics.setup_physics_simulation` after reading the physics wiki:

- Call: `physics.setup_physics_simulation` with `meshPath` = the Manny skeletal
  mesh (`SKM_Manny`), `physicsAssetName: PA_Manny_Ragdoll`,
  `savePath: /Game/Physics`, `assignToMesh: true`.
- Result: the call and 4 retries all failed the liveness ping; a subsequent
  trivial `actor.list {}` failed identically; `/Game/Physics` never appeared on
  disk (no asset created). Editor PID stayed alive/Responding at ~37% CPU with a
  frozen log — a wedge, not a crash (no crash dump written for this session; the
  active editor log has no `Assertion failed` / `Fatal error` / callstack).

Editor-log wedge line (last line written before the game thread stopped making
progress; the log has been frozen at this line for 11+ minutes while the process
stays alive), from `Saved/Logs/EAContentExamples57.log`:

```
[2026.07.04-21.00.53:859][975]LogStreaming: Display: FlushAsyncLoading(401): 1 QueuedPackages, 0 AsyncPackages
```

(No shutdown / "Log file closed" / exited line follows it — the process is
wedged, not cleanly stopped.)

## What it should do

Create the physics asset headlessly, with no modal. Options, cheapest first:

1. Reuse the plugin's own mesh-free generator idea: call the non-interactive
   engine helper `FPhysicsAssetUtils::CreateFromSkeletalMesh(PhysicsAsset, Mesh,
   FPhysAssetCreateParams{}, ...)` on a freshly `NewObject<UPhysicsAsset>`,
   bypassing `UPhysicsAssetFactory` entirely (the factory only exists to wrap that
   call with a UI). The bare-skeleton branch already demonstrates the
   "build-bodies-directly, no factory" shape.
2. If the factory is kept, gate the modal off: it is only shown when the app is
   interactive; the MCP context should force the unattended / no-dialog code path
   (e.g. run under a scope that makes `FApp::IsUnattended()` true, or use a factory
   entry point that skips `EditorAddModalWindow`).

This mirrors the fixes on the sibling modal-blocker tickets (see below), which
resolve headless hangs by bypassing the interactive dialog and driving the
underlying non-interactive API directly.

severity rationale: impact=game-thread-hang (a normal documented call wedges the
entire editor session with no error and no asset; the editor must be OS-killed and
every subsequent RPC in the session times out) x reach=any mesh-backed
physics-asset creation, a documented primary ragdoll-setup path exercised by two
sibling verbs (not a rare edge) -> High. (A hang, not a crash/corruption, so not
Critical.)

## Distinct from / see also

- `B-create-physics-asset-skeletonpath-alias-dead` (IN-REVIEW) — a schema
  ALIAS-registration bug on `skeleton.create_physics_asset` (`skeletonPath` dead,
  rejected with `MISSING_REQUIRED_PARAM` before the body runs). Different layer,
  different root cause; its positive replay used the bare-skeleton branch and so
  never hit this factory modal.
- `B-merge-actors-rejects-valid-selection` (OPEN) — same systemic pattern
  (`CreateModalSaveAssetDialog` blocks the headless game thread) in a different
  verb; resolved by writing the package directly, bypassing the modal. Precedent
  for the fix here.
- `B-create-pose-library-noop-fake-success` — notes `ConfigureProperties` popping
  a modal asset picker plus `FactoryCreateNew` on a different verb.
- `B-create-anim-blueprint-duplicate-name-crash` — `UAnimBlueprintFactory::FactoryCreateNew`
  in the MCP context, but a CRASH (duplicate-name assert), not a modal hang.

## History

- `#1-initial-repro` `OPEN` reporter — Editor_down iteration: a physics/ragdoll
  authoring attempt read the physics wiki, then issued
  `physics.setup_physics_simulation` on the Manny skeletal mesh
  (`SKM_Manny` -> `PA_Manny_Ragdoll`, `/Game/Physics`, `assignToMesh:true`). That
  first game-thread RPC wedged the editor: the call plus 4 retries and a trivial
  `actor.list {}` all timed out on the liveness ping, no `/Game/Physics` asset was
  created, and the editor stayed alive/Responding at ~37% CPU with a frozen log
  and no crash dump. Canary `call()` (namespace index) also failed twice after a
  ~40s backoff — no recovery. Traced by source read to the shared
  `UPhysicsAssetFactory` mesh-backed path in `PhysicsHandler.cpp:421-426`
  (`AssetTools::CreateAsset`) and `PhysicsAssetHandler.cpp:315-325`
  (`Factory->FactoryCreateNew`), which opens the interactive body-generation modal
  on the game thread in this non-`-unattended` editor. Filed as a family hang
  spanning both physics-asset-creation verbs (mesh-backed only; the bare-skeleton
  branch is unaffected).
