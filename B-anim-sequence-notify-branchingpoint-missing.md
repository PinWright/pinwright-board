---
id: B-anim-sequence-notify-branchingpoint-missing
title: "anim_sequence.json notify entries missing 'branchingPoint' flag"
status: DONE
severity: Low
category: bug
tags: [anim-sequence, sidecar]
---

# anim_sequence.json notify entries missing branchingPoint flag

`AnimSequenceDumpBuilder.cpp` already emits `branchingPoint` per notify, including `false` values, and the formatter theory was wrong. The missing behavior was stale sidecar cache invalidation: `anim_sequence.json` had no explicit aspect version, so old default-version dumps could remain fresh after the notify schema changed.

Result: consumers can't tell which notifies fire as branching points vs regular notifies.

## Sample

`App/Meshes/Drone/SK_Drone2/anim_sequence.json` — notifies array empty in this asset, so the useful regression surface is any populated notify array plus cache metadata that forces stale sidecars to regenerate.

## Fix sketch

- Add an explicit `anim_sequence.json` aspect version bump in `AssetDumpCache.cpp`.
- Cover that version through `GetAspectVersion()` and `MakeCurrentAspectVersions()` regression assertions.
- Keep the existing anim sequence dump tests pinned to populated notify JSON that includes `branchingPoint: false`, including the on-disk `anim_sequence.json` sidecar.

## History
- `#1-branchingpoint-missing` `OPEN` reporter — minor schema gap; required for understanding montage-style notify behavior.
- `#2-version-anim-sequence-aspect` `IN-REVIEW` developer — Added an anim_sequence.json aspect version bump so stale cached sidecars are invalidated, and added regression coverage for the version plus on-disk notify branchingPoint=false output.
- `#3-verify-branchingpoint-emitted` `DONE` tester — Verified: ran asset.dump on /App/Meshes/Robot/RobotArm_01; freshly written anim_sequence.json shows both notify entries with "branchingPoint": false (N_Sparks_ON @ 1.0018, N_Sparks_OFF @ 1.5925).
