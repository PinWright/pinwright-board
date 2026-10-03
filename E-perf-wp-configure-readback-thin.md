---
id: E-perf-wp-configure-readback-thin
title: "performance.* CVar-write verbs answer a bare string or echo the request: read the values back the way set_scalability and apply_baseline_settings already do"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, performance, world-partition, texture-streaming, datalayer, readback, verify-after-mutate, wiki, cvar-write-no-echo]
encounters: 2
lastSeen: 2026-07-09T11:25:33.2635718+03:00
rice: [2, 1, 1, 2]
priority: 8
---

# Most `performance.*` setters cannot confirm their own write

Nine `performance.*` verbs write engine settings and report nothing the engine read back
(`Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp`):

| verb | writes | response |
|---|---|---|
| `set_resolution_scale` | `r.ScreenPercentage` | `"Resolution scale set"` (`:485`) |
| `set_vsync` | `r.VSync` | `"VSync configured"` (`:503`) |
| `set_frame_rate_limit` | `GEngine->SetMaxFPS` | `"Max FPS set"` (`:521`) |
| `configure_nanite` | `r.Nanite` | `"Nanite configured"` (`:538`) |
| `configure_lod` | `r.MipMapLODBias`, `r.ForceLOD` | `"LOD settings configured"` (`:569`) |
| `configure_texture_streaming` | `r.Streaming.PoolSize`, `r.TextureStreaming` | `"Texture streaming configured"` (`:614`) |
| `enable_gpu_timing` | `r.GPUStatsEnabled` | request echo `enabled` (`:1145`) |
| `optimize_draw_calls` | `r.MeshDrawCommands.*` | request echoes `optimized`, `instancing` (`:1260-1261`) |
| `configure_occlusion_culling` | `r.AllowOcclusionQueries`, `r.OcclusionSlop`, `r.OcclusionCullMinScreenRadius` | request echoes (`:1305-1309`) |

Every CVar write goes through `CVarPriorityPreservingSet::SetPreservingPriority`, which silently does
nothing when the CVar is not found and whose own header says it is "NOT a substitute for reading
the value back" because clamps and `OnChanged` sinks still apply
(`Source/PinWright/Private/Handlers/CVarPriorityPreservingSet.h:41-58`). Two siblings already do it
right: `set_scalability` re-reads each `sg.*` group (`PerformanceHandler.cpp:447`, reported as
`appliedGroups` `:457`) and `apply_baseline_settings` re-reads each CVar (`:1183`, `appliedCVars`
`:1220`). Replay evidence: `configure_texture_streaming {poolSize:800}` returned only the bare string,
and confirming the pool took (1536 -> 800) needed a separate `system.console.search`.

**Workaround:** read each CVar with `system.console.search` / `system.console.get` after the call
(documented for texture streaming at `docs/wiki-src/performance.md:74`).

**Fix:** have the nine verbs return `appliedCVars: [{cvar, value}]` re-read after the write, using the
`apply_baseline_settings` entry helper, with a `found:false` entry (or a warning) for a CVar that does
not exist; `set_frame_rate_limit` reports `GEngine->GetMaxFPS()`. Keep the existing `message` /
echo fields for compatibility.

**Acceptance:** `configure_texture_streaming {poolSize:800}` returns
`appliedCVars` containing `{cvar:"r.Streaming.PoolSize", value:800}`; `set_vsync {enabled:false}` on
a host where `r.VSync` is held at `ECVF_SetByGameSetting` reports the value that survived; each of
the nine verbs returns `appliedCVars` (or `maxFps`) in its response.

## Related

- `B-configure-world-partition-silent-noop` (IN-REVIEW) — a different verb that wrote nonexistent
  CVars.
- `E-viewport-info-no-render-resolution` — wants the `set_resolution_scale` read-back for its own
  reason.
- `B-performance-typed-verbs-pin-scalability-cvars` (IN-REVIEW) — introduced the priority-preserving
  write these verbs use.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a
  `performance.configure_world_partition` open-world streaming task whose story
  explicitly asked the agent to "report back what each call returned so I know
  ... it actually took." Distinct PROCESS/ergonomic angle from the judge-filed
  `B-configure-world-partition-silent-noop` (that ticket is the silent-no-op
  echo defect on one method; this is the read-back thinness on its two siblings).
  Both verbs return bare success strings with no value echo: handler-confirmed at
  `PerformanceHandler.cpp:290` (`"Texture streaming configured"`, no
  enabled/poolSize/boost echo; CVar `Set` guarded by `if(CVar)` at :267-269/:286-288
  with no found/not-found signal) and `WorldPartitionHandler.cpp:145`
  (`"DataLayer '<name>' created."`, no asset-path/field echo). Friction note
  (verbatim): "configure_texture_streaming/create_datalayer return bare
  'configured/created' messages without echoing applied values ... the newly
  created Foliage_Streaming layer does not appear in
  get_level_structure_info.dataLayers." Net: with the returns this thin and the
  only structure read-back coming back with an empty `dataLayers` array, the
  story's verify ask was unanswerable from any surface. Same readback-thinness
  shape as the OPEN `E-audio-get-info-soundclass-mix-readback-thin` /
  `E-audio-authoring-attenuation-readback-undocumented`, here on
  `performance` + `world_partition`. Proposed: add verify-after-mutate notes to
  `docs/wiki-src/performance.md` and `docs/wiki-src/world_partition.md` naming the
  bare returns and the live confirm paths (`system.console.*` on
  `r.Streaming.PoolSize`; data-layer listing/`asset.dump` for the layer);
  optionally widen both returns to echo observed state. Dedup: ripgrep + qmd
  across OPEN/closed found no ticket on the `configure_texture_streaming` or
  `create_datalayer` return-thinness (the only adjacent ticket,
  `B-configure-world-partition-silent-noop`, is a different method and a
  silent-no-op bug, not a readback-ergonomics gap).
- `#2-additional-family-scope` `OPEN` reporter — Additional evidence (broader
  scope, replay-confirmed live at HEAD): re-hit directly from a
  `performance.configure_texture_streaming` seed (raise the texture-streaming
  pool to ~1.5 GB on a large open-world level, then confirm the pool actually
  took effect — not just that the call returned OK). Replay via
  `mcp__pinwright__call`: `configure_texture_streaming {enabled:true,
  poolSize:800, boostPlayerLocation:true}` returned the same bare
  `{message: "Texture streaming configured"}` — no echo of
  enabled/poolSize/boostPlayerLocation — so a follow-up
  `system.console.search r.Streaming.PoolSize` was required to confirm
  currentValue flipped 1536 -> 800. NEW ANGLE: the bare-string-no-echo shape is
  NOT confined to `configure_texture_streaming` + `create_datalayer` — it spans
  the WHOLE `performance` CVar-write verb family. Current source
  `PerformanceHandler.cpp`: `set_resolution_scale` (:472 "Resolution scale
  set"), `set_vsync` (:488 "VSync configured"), `set_frame_rate_limit` (:506
  "Max FPS set"), `configure_nanite` (:522 "Nanite configured"), `configure_lod`
  (:551 "LOD settings configured"), `configure_texture_streaming` (:591 "Texture
  streaming configured") all `SendSuccess` a flat literal with no readback. By
  contrast two siblings in the SAME namespace already read the CVars back and
  report effective values: `set_scalability` (:445-451 returns
  requestedLevel/appliedGroups/requestedLevelApplied bool, documented "so a
  success can never silently mean 'nothing changed'") and
  `apply_baseline_settings` (per its wiki "reports the exact CVar names and
  values applied") — those two are the reference implementation the rest of the
  family should copy. Replay also settles the #1 silent-no-op fear on the normal
  path: `CVar->Set((float)PoolSize)` uses the default `ECVF_SetByCode` priority
  (2nd-highest), so the write DID land in replay (1536 -> 800) over the
  Scalability-flagged CVar — the harm here is purely the missing echo, not an
  actual no-op (the real no-op tracked in `B-configure-world-partition-silent-noop`
  is a different method that targets nonexistent CVars). Also seen this run but
  NOT filed: `system.inspect.get_memory_stats` is a clean, intentional
  NOT_IMPLEMENTED stub whose wiki redirects to the `performance.*` memreport/stat
  verbs — well-documented discoverability, not a defect. Fix unchanged: widen the
  family returns to echo the read-back CVar values (mirror `set_scalability`).
- `#3-rephrased` `OPEN` developer — The `world_partition.create_datalayer` half is fixed: its success branch now returns `dataLayerName` and `dataLayerAssetPath` (`Source/PinWright/Private/Handlers/World/WorldPartitionHandler.cpp:189-193`), and the docs ask for `configure_texture_streaming` is done (`performance.md:74` says it is an acknowledgement and names the console read-back). Retitled and rewritten to the `performance.*` CVar-write family per `#2`: six bare-string setters (now `PerformanceHandler.cpp:485,503,521,538,569,614`) plus three request-echo verbs (`:1145`, `:1260-1261`, `:1305-1309`), with `set_scalability` / `apply_baseline_settings` as the reference; dropped the datalayer and `get_level_structure_info` narrative and the stale `:290` citations, added Acceptance. Severity unchanged (Low).
