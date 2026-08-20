---
id: B-landscape-get-heights-dirties-map
title: "landscape.get_heights dirties the map package — a pure read leaves the level with unsaved changes"
status: OPEN
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
consequence. `FLandscapeEditDataInterface` is an **edit** interface: constructing one takes
the landscape's edit lock and its destructor flushes, and that path marks components /
their packages dirty whether or not any height was written. `bUploadTextureChangesToGPU =
false` suppresses the GPU upload; it does not make the object read-only.

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

## Not yet checked

Whether the sibling read verbs that use the same interface (`landscape.get_weights` and
any other `FLandscapeEditDataInterface`-based reader) have the same behaviour. They almost
certainly do; the same control run would settle it in a minute.
