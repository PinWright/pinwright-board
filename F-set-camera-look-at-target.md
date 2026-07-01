---
id: F-set-camera-look-at-target
title: "editor.set_camera has no look-at target; agents must hand-compute pitch/yaw for 'aim at X' intents"
status: OPEN
severity: Low
category: feature
tags: [editor, viewport, set_camera, camera, look-at, ergonomic, docs]
encounters: 1
lastSeen: 2026-06-17T10:29:49Z
---

# editor.set_camera has no look-at target; agents must hand-compute pitch/yaw for "aim at X" intents

`editor.set_camera` accepts only an explicit `location {x,y,z}` and `rotation {pitch,yaw,roll}` (`ViewportHandler.cpp:165-179`; `RPC_PARAM_OPT("location"...)`, `RPC_PARAM_OPT("rotation"...)`). But the natural way a user frames a viewport shot is by where the camera *looks*, not by Euler angles — "position the camera at (1200,-800,600) **looking back toward the level origin**", "a lower-angle vantage at (-400,600,200)". To satisfy that, the agent must manually do the look-at trig (`atan2` for yaw, `atan2(dz, horizontal_dist)` for pitch) to turn "camera at A looking at B" into a `rotation` field, for every such shot.

This is pure boilerplate the tool could absorb. Proposed: add an optional `lookAt {x,y,z}` (world-space focus point) parameter to `editor.set_camera`; when present and `rotation` is omitted, derive the rotation from `(lookAt - location)` via `FRotationMatrix::MakeFromX(...).Rotator()` (or `UKismetMathLibrary::FindLookAtRotation`) and apply it. `rotation` stays supported and wins if both are given. Optionally accept a `distance`/`pitch`/`yaw` orbit form too, but `lookAt` alone covers the common establishing-shot case.

This is a PROCESS-friction finding, not a tool defect — the task completed cleanly (all 17 calls `ok`, outcome `clean`). It is logged so the recurring "compute look-at toward a point" manual step shows up across tasks; cinematic/walkthrough/review stories repeatedly express camera placement as look-at, so the same hand-trig recurs.

**Workaround:** compute the rotation client-side: `yaw = atan2(dy, dx)`, `pitch = atan2(dz, sqrt(dx^2+dy^2))` (degrees), `roll = 0`, where `(dx,dy,dz) = lookAt - location`; pass the result as `rotation`.
**Fix:** add the optional `lookAt` param + server-side look-at derivation (and document it on the `editor.set_camera` overlay in `docs/wiki-src/editor.md`, which currently lists the method but not its `location`/`rotation` signature or any look-at affordance).

## History
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of an `editor`-namespace cinematic-walkthrough task (17 calls, all ok, outcome `clean`). Self-reported friction note: "only extra step was computing look-at pitch/yaw toward origin ... both expected." The agent did this look-at trig **twice** — `set_camera loc(1200,-800,600) rot(-22.59,146.31,0)` (frame toward origin) and `set_camera loc(-400,600,200) rot(-15.5,-56.31,0)` (lower-angle vantage) — because `editor.set_camera` exposes only explicit `location`/`rotation` (`ViewportHandler.cpp:165-179`), no look-at/target convenience. Dedup checked: distinct from `B-set-camera-no-viewport-redraw` (redraw flush, DONE) and `E-viewport-info-camera-transform` (camera read-back, DONE) — this is a write-side input-ergonomics gap; no existing camera/look-at ticket on the board.
