---
id: F-geometry-uv-prep-pipeline-batch
title: "No composite/batch UV-prep op — the canonical unwrap→project→transform→pack pipeline is 4 sequential same-actor calls"
status: OPEN
severity: Low
category: feature
tags: [geometry, uv, unwrap_uv, project_uv, transform_uvs, pack_uv_islands, batch, pipeline, docs]
encounters: 2
lastSeen: 2026-06-23T10:08:23Z
---

# The canonical "prep the UVs end to end" intent decomposes into 4 separate same-actor UV ops

UV preparation on a DynamicMeshActor is a recurring authoring intent that the
user naturally expresses as **one** conceptual step ("prep the UVs end to end",
"clean up the UVs so the texture artist can paint it"). It decomposes into a
fixed chain of four `geometry` methods that all operate on the **same actor and
the same UV channel**, in order, each consuming the previous step's result:

- `geometry.unwrap_uv` (`actorName`, `uvChannel`) — XAtlas auto-unwrap.
- `geometry.project_uv` (`actorName`, `projectionType`, `scale`, `uvChannel`) — re-project (box/planar/cylindrical).
- `geometry.transform_uvs` (`actorName`, `uvChannel`, `translateU/V`, `scaleU/V`, `rotation`) — seat the islands in the unit square.
- `geometry.pack_uv_islands` (`actorName`, `uvChannel`, `textureResolution`) — pack for a target atlas.

Every one of these repeats `actorName` and `uvChannel`; the only per-step
information is the handful of op-specific knobs. There is no composite verb that
takes the actor + channel once and runs an ordered list of UV stages, so the
canonical pipeline is always N separate RPCs (here 4), with the actor/channel
restated each time.

## Why this is friction (not just normal granularity)

This is the same class of finding as `F-geometry-lod-settings-batch` (a single
canonical authoring intent — "set up the LOD ladder" — that costs N sequential
same-surface calls). Here the intent is "prep the UVs"; it costs four ordered
calls that differ only in which UV stage they run, with `actorName`+`uvChannel`
duplicated across all four. The cost is paid every time a mesh's UVs are
authored end to end, and the ordering (unwrap before project before pack) is
implicit knowledge an agent must already hold — there is no single entry point
that encodes the canonical order or lets the agent express the pipeline as one
declarative request.

This is a clean-outcome PROCESS finding: every call in the task succeeded first
try with no retries, errors, workarounds, or `python.execute` fallback (see
evidence). The friction is the call *count* and the repeated actor/channel
boilerplate on an otherwise clean run, surfaced by the struggle audit (the
per-finding judge filed nothing — `filed_id` empty).

## What it should do

Pick one of:
1. **Composite handler (preferred, matches `F-geometry-lod-settings-batch` /
   `F-batch-pin-defaults` batch precedent):** add a `geometry.prep_uvs` (or
   `geometry.apply_uv_pipeline`) that takes `actorName` + `uvChannel` **once**
   plus an ordered `stages: [{op: "unwrap"|"project"|"transform"|"pack", ...op-params}]`
   array, applied under one transaction with a per-stage `results` array. The
   four existing singular methods stay for fine-grained use. This collapses the
   canonical pipeline from 4 calls to 1 and makes the ordering explicit in the
   request shape.
2. **Docs-only (minimum):** document the canonical UV-prep order on the
   `geometry` overlay (`docs/wiki-src/geometry.md`) — a short "UV-prep pipeline"
   note listing unwrap → project → transform → pack with the rationale for the
   order, and reciprocal `## See also` links between the four method pages — so
   an agent reaching any one UV op learns the full intended sequence without
   reconstructing it. (The downstream wiki edit is the wiki-authoring process,
   not this audit's job; this ticket names the page.)

## Friction evidence (this task — geometry.pack_uv_islands "StonePillar" build, 10 calls, outcome clean)

Story: build a StonePillar cylinder (r60/h300/24seg), add topology
(subdivide+bevel), then **prep the UVs end to end** on channel 0. The UV-prep
half of the call log is the fixed four-step pipeline, every step on the same
actor/channel, all `ok:true`:

- `geometry.unwrap_uv {StonePillar, uvChannel 0 (XAtlas)}` → ok
- `geometry.project_uv {StonePillar, cylindrical, scale 1, uvChannel 0}` → ok
- `geometry.transform_uvs {StonePillar, uv0, scale 0.9, translate 0.05}` → ok
- `geometry.pack_uv_islands {StonePillar, uvChannel 0, textureResolution 2048}` → ok

The agent's friction note was *"none — wiki listed every geometry method and its
params clearly; all 8 RPCs succeeded first try with no retries or fallbacks."* So
nothing errored — this is pure call-count / repeated-boilerplate overhead on an
otherwise clean run. Final `geometry.get_mesh_info` confirmed a valid single
mesh (434 verts, 864 tris, hasUVs:true).

## Docs page to improve

`docs/wiki-src/geometry.md` — add a "UV-prep pipeline" note (canonical
unwrap → project → transform → pack order + rationale) and reciprocal
`## See also` cross-references across the four UV method pages, so the intended
sequence is visible at the point any single UV op is discovered.

## History
- `#2-recurrence-stonewell-unwrap-pack-subset` `OPEN` reporter — Recurrence (second task) exercising the **unwrap→pack subset** of the pipeline (no project/transform stage). Struggle audit of the `geometry.convert_to_static_mesh` "StoneWell" build (15 calls, outcome clean, friction:"none", all RPCs first-try clean, judge filed nothing). The story's UV step ("Auto-generate UV coordinates (XAtlas unwrap) and pack the UV islands so it's ready to texture") decomposed into the canonical pipeline's bookend stages, both on the same actor/channel: `geometry.unwrap_uv {StoneWell, uvChannel 0 (XAtlas)}` → ok, then `geometry.pack_uv_islands {StoneWell, uvChannel 0, textureResolution 1024}` → ok. `actorName`+`uvChannel` restated across both calls; no project/transform here, so this is the 2-stage minimal form of the pipeline, not the full 4. Confirms the call-count/boilerplate friction recurs even when only the unwrap and pack ends of the pipeline are needed — a `geometry.prep_uvs` composite (or the docs-only canonical-order note) with `stages:[{op:"unwrap",...},{op:"pack",...}]` would still collapse the unwrap+pack subset to one declarative call. Same proposal as #1; distinct from `E-geometry-auto-uv-redundant-with-unwrap-uv` (the auto_uv/unwrap_uv discovery-tax pair, not exercised here — the agent reached unwrap_uv directly) and from `E-geometry-deformer-echo-mesh-counts` (deformer count-echo, appended separately for this same task's shell/noise_deform readback).
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.pack_uv_islands` "StonePillar" task (10 calls, outcome clean, judge filed nothing, `filed_id` empty). PROCESS finding: the "prep the UVs end to end" intent decomposes into 4 ordered same-actor/same-channel ops — `geometry.unwrap_uv`, `geometry.project_uv`, `geometry.transform_uvs`, `geometry.pack_uv_islands` (signatures confirmed in wiki-generated/geometry.{unwrap_uv,project_uv,transform_uvs,pack_uv_islands}.md; all four take `actorName`+`uvChannel`) — with no composite entry point, so `actorName`/`uvChannel` are restated 4× and the canonical order is implicit. No retries/errors/fallbacks; agent friction note "none". Proposed: add a `geometry.prep_uvs` composite with an ordered `stages:[{op,...}]` array (mirrors `F-geometry-lod-settings-batch` / `F-batch-pin-defaults` batch precedent), or at minimum document the canonical pipeline order + reciprocal See-also links on `docs/wiki-src/geometry.md`. Dedup: distinct from `E-geometry-auto-uv-redundant-with-unwrap-uv` (that's two redundant XAtlas verbs / discovery tax, not a multi-op pipeline) and from `F-geometry-lod-settings-batch` (same friction class but the LOD surface, not UV prep); no existing UV-pipeline/batch ticket on the board.
