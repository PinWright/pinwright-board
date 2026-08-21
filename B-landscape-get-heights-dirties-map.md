---
id: B-landscape-get-heights-dirties-map
title: "landscape.get_heights dirties the map package — a pure read leaves the level with unsaved changes"
status: IN-REVIEW
severity: Medium
category: bug
tags: [landscape, get_heights, read-only, dirty-package, false-dirty, measurement, LandscapeEditDataInterface]
encounters: 1
lastSeen: 2026-08-21T01:30:00+05:00
---

# `landscape.get_heights` marks the level dirty

`landscape.get_heights` is the READ counterpart to `landscape.sculpt` / `landscape.edit`
and its own registration string calls it "Read back a landscape's heightmap over a region".
It changes nothing. It nevertheless leaves the map package dirty.

## Reproduction, with a control

Measured on `EAContentExamples58` / `Dota2_Map_688`, 2026-08-21:

1. Fresh editor boot, level opened, nothing touched.
   `editor.list_dirty_packages` -> `count: 0`.
2. Exactly one `landscape.get_heights` over a **5x5** region, `includeSamples` true.
   No other RPC in between.
3. `editor.list_dirty_packages` -> `count: 1`, `/Game/Maps/Dota2_Map_688`.

A 5x5 read is small enough that nothing else plausibly ran. The step from 0 to 1 is the
read.

## Guilty source

`Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp`, in the
`landscape.get_heights` handler registered at `:1729`:

```cpp
// Read-only interface (bUploadTextureChangesToGPU=false): this is the same GetHeightData
// read the write path already performs (the read half of landscape.edit's read-modify-write),
// so no new access mechanism is introduced.
TArray<uint16> Heights;
Heights.SetNumZeroed(RegionSize);
FLandscapeEditDataInterface LandscapeEditRead(LandscapeInfo, false);
LandscapeEditRead.GetHeightData(MinX, MinY, MaxX, MaxY, Heights.GetData(), 0);
```

The comment is right that no new access mechanism is introduced and wrong about the
consequence. `FLandscapeEditDataInterface` is an **edit** interface, and
`bUploadTextureChangesToGPU = false` suppresses the GPU upload; it does not make the
object read-only.

**Corrected mechanism (measured against UE 5.8 source while fixing, see `#2`).** It is
neither the constructor's edit lock nor the destructor's flush — both were guesses and
both are wrong. It is a single call on the way to the pixels:

`GetHeightData` → `GetHeightDataTempl` (`LandscapeEditInterface.cpp:1358`) →
`GetHeightDataInternal` (`:859`) → `GetHeightMapColor` (`:815`) →
`FLandscapeTextureDataInterface::GetTextureDataInfo` (`:3451-3457`), which constructs an
`FLandscapeTextureDataInfo` whose constructor calls
`Texture->Modify(bShouldDirtyPackage)` (`:4028-4043`) with the interface's
`bShouldDirtyPackage` defaulting to **true** (`:88-90`). The heightmap texture is outered
to the landscape proxy actor (`ALandscapeProxy::CreateLandscapeTexture`,
`LandscapeEdit.cpp:7954-7957`), so `UObject::Modify` (`Obj.cpp:1652-1680`) marks the
**proxy's** package — the map package on a classic level, which is exactly what the
control run observed.

The engine's own comment at `:4036-4038` admits the `Modify()` is unwanted for read-only
use and that removing it properly would mean rebuilding `TLandscapeEditCache`. It
therefore ships the opt-out instead: `FLandscapeTextureDataInterface::SetShouldDirtyPackage(false)`
(`LandscapeEdit.h:91`), which is the fix.

## Why it matters

The dirty flag is this project's ONLY reliable evidence that a write landed. The house rule
is to cross-check `editor.list_dirty_packages` against
`EditorLoadingAndSavingUtils.get_dirty_map_packages()` and treat disagreement as a failed
write, because `clear_instances`/`add_instances` famously do NOT dirty and a `level.save`
after them silently no-ops. A read verb that dirties destroys that signal in both
directions:

- **False positive.** After any measurement pass the level looks modified. An agent that
  saves "because it is dirty" writes a package it never meant to change; an agent that
  quits gets an unsaved-changes prompt and has to choose discard, which is exactly the
  moment real work gets thrown away by mistake.
- **The check stops discriminating.** If reading dirties, "the map is dirty" no longer
  distinguishes a successful apply from a no-op apply that happened to be preceded by a
  read-back. The verification recipe in the project's own `CLAUDE.md` depends on it doing so.

Encountered while snapshotting all 1,145,215 heightmap samples of `Dota2_Map_688` for an
offline ground-Z sampler. The run was strictly read-only and the level had to be quit with
`discard: true` twice to avoid writing a package nothing had changed.

## Suggested fix

Read without an edit interface, or restore the dirty state around the read.

- Preferred: read the height data through a genuinely const path so no edit lock is taken.
- Failing that, snapshot `UPackage::IsDirty()` for every package the interface could touch
  (the landscape actor's, its proxies', and the map's) before constructing
  `FLandscapeEditDataInterface`, and restore each with `SetDirtyFlag` afterwards — noting
  that `EditorAssetSubsystem.set_dirty_flag` refuses any package containing a map, so the
  restore has to go through `UPackage` directly.
- Whichever is chosen, the handler should assert its own postcondition in a test: dirty
  count before == dirty count after, on a clean-boot level.

## Siblings — checked

There is no `landscape.get_weights` verb; `get_heights` is the only pure read verb built
on this interface. The other four `FLandscapeEditDataInterface` reads in
`LandscapeHandler.cpp` are read *halves inside write verbs*, and every one of them shared
the defect (`GetWeightDataFast` reaches the same `GetTextureDataInfo` →
`Texture->Modify` path at `LandscapeEditInterface.cpp:2650,2662`):

- `landscape.sculpt` pre-edit snapshot — mattered on its own, because a sculpt refused
  with `LANDSCAPE_SCULPT_NO_CHANGE` changed nothing and still left the level dirty.
- `landscape.sculpt` verification readback.
- `landscape.edit` read half of the read-modify-write.
- `landscape.create_procedural_terrain` weightmap verification readback.

All four now take the same opt-out. Safe because all three write verbs dirty explicitly
and independently (`Landscape->MarkPackageDirty()` in `sculpt` / `edit`,
`PinWright::MarkLevelActorModified` in the paint verb), so a non-dirtying read cannot
silence a real write.

## History

- `#1-initial-repro` `OPEN` reporter — "5x5 `landscape.get_heights` with `includeSamples` on a fresh boot of `EAContentExamples58` / `Dota2_Map_688` takes `editor.list_dirty_packages` from count 0 to count 1, `/Game/Maps/Dota2_Map_688`, with no other RPC in between. Hit while snapshotting 1,145,215 heightmap samples for an offline ground-Z sampler; the level had to be quit with `discard: true` twice."
- `#2-opt-out-of-texture-modify` `IN-REVIEW` developer — "Root cause corrected in the body above: not the edit lock and not the destructor's flush, but `FLandscapeTextureDataInfo`'s constructor calling `Texture->Modify(bShouldDirtyPackage)` with the interface default of true (`LandscapeEditInterface.cpp:4028-4043`, `:88-90`), reached from `GetHeightData` via `GetTextureDataInfo` (`:3451-3457`); the heightmap texture is outered to the landscape proxy, so the flag lands on the map package. Fixed in `Handlers/Environment/LandscapeHandler.cpp` by calling the engine's own `SetShouldDirtyPackage(false)` on every read-only `FLandscapeEditDataInterface` — `get_heights` plus the four sibling read halves listed above — and, for `get_heights` only, wrapping the read in a new `PinWright::PackageDirty::FScopedPackageDirtyRestore` (`Utils/PackageDirtyUtils.h`) that captures and restores the dirty flags of the landscape actor, every proxy and the level, and logs a Warning naming the package if it ever actually has to restore one, so a surviving cause is reported rather than masked. Contract is preserve, not clear. Tests `PinWright.landscape.get_heights.CleanLevelStaysClean` and `.DirtyLevelStaysDirty` assert both starting states over a code-built fixture landscape. `Docs/rpc-design.md` §11 and `Docs/wiki-src/landscape.md` updated in the same commit. Plugin commit `faac6950`. Compile-checked with `-SingleFile` on all three touched `.cpp` files (`Result: Succeeded`, real `[1/1] Compile` lines); the full build and automation suite are owned by a peer agent this wave and have NOT been run here."
