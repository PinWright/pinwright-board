---
id: E-perf-wp-configure-readback-thin
title: "performance.configure_texture_streaming and world_partition.create_datalayer return bare 'configured/created' strings with no value echo or readback, so a task that asks 'tell me what each call returned' cannot confirm the write"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, performance, world-partition, texture-streaming, datalayer, readback, verify-after-mutate, wiki, cvar-write-no-echo]
encounters: 2
lastSeen: 2026-07-09T11:25:33.2635718+03:00
---

# `configure_texture_streaming` / `create_datalayer` confirm with a bare message, not the applied values

Two mutate verbs in the streaming-setup workflow return a flat success **string**
and nothing else, so an agent that follows the documented "do it, then read back
what it returned" verify pattern gets no value to confirm against. This is the
same readback-thinness shape as `E-audio-get-info-soundclass-mix-readback-thin`
(OPEN, `docs`) and `E-audio-authoring-attenuation-readback-undocumented` (OPEN,
`docs`) — here scoped to the `performance` texture-streaming verb and the
`world_partition` data-layer verb.

This is **distinct from** the judge-filed bug
`B-configure-world-partition-silent-noop` (which is about
`performance.configure_world_partition` echoing inputs it never actually applied
because it targets nonexistent CVars). That ticket is the silent-no-op defect on
one method; **this** ticket is the broader read-back ergonomics gap on its two
sibling methods in the same task that return *no* values at all — not even an
echo — and whose effects can't be confirmed through any documented surface.

## The two thin returns (handler-confirmed)

- **`performance.configure_texture_streaming`** — `PerformanceHandler.cpp:290`
  returns the bare literal `SendSuccess("Texture streaming configured")`. It
  echoes none of `enabled` / `poolSize` / `boostPlayerLocation`, and — like the
  WP-configure no-op — it sets `r.Streaming.PoolSize` / `r.TextureStreaming`
  behind `if (CVar)` guards (`:267-269`, `:286-288`) and silently skips when the
  CVar isn't found, with no JSON signal of whether the pool size actually took.
  So `{enabled:true, poolSize:1500, boostPlayerLocation:true}` returns the same
  string as `{}` — the caller cannot tell a 1500 MB pool from a no-op.
- **`world_partition.create_datalayer`** — `WorldPartitionHandler.cpp:145`
  returns `SendSuccess("DataLayer '<name>' created.")`. It does not echo the
  created instance's data-layer **asset path**, short name, or any field a caller
  could use to address or verify the layer afterward.

## Why it matters — the process friction (this task)

The seed story (`performance.configure_world_partition`, open-world streaming
setup) ended with an explicit verify demand: *"Walk me through each step and
report back what each call returned so I know the cell size and loading range
actually took."* The agent dutifully called both verbs and got back only
`"Texture streaming configured"` and `"DataLayer 'Foliage_Streaming' created."`
— success strings with nothing to report. There is nothing to "report back" for
either, so the story's core ask (confirm the writes) is unanswerable from the
return values alone.

The agent then reached for the one available read-back —
`level.structure.get_level_structure_info` — as a post-config confirmation, and
its `dataLayers` array came back **empty**, so even the indirect verify path did
not show the just-created `Foliage_Streaming` layer. Friction note (verbatim):

> "configure_texture_streaming/create_datalayer return bare 'configured/created'
> messages without echoing applied values ... the newly created Foliage_Streaming
> layer does not appear in get_level_structure_info.dataLayers."

So between the thin returns and the empty structure read-back, the agent had **no
way to confirm** the texture pool or the data layer landed — every confirmation
surface was either a bare string or empty. (Whether `get_level_structure_info`
*should* list the new layer is a separate tool-bug question for the judge; the
PROCESS point here is that with the returns this thin, that structure read is the
*only* fallback verify path, and it didn't help either.)

## What it should do / how to fix (docs-first, NAMES the overlay pages)

The overlay edit is a downstream wiki process, not this audit's job — naming the
pages and the change is the deliverable. Both overlays
(`docs/wiki-src/performance.md`, `docs/wiki-src/world_partition.md`) are today
one-line namespace blurbs with no verify-after-mutate guidance.

- **`docs/wiki-src/performance.md`** — add a short "verify-after-mutate" note
  that `configure_texture_streaming` returns only a confirmation string and does
  **not** echo the applied pool size or report whether the underlying CVars were
  found; point to `system.console.get`/`system.console.search` on
  `r.Streaming.PoolSize` / `r.TextureStreaming` as the live confirm path for the
  pool/enable state (and reference `B-configure-world-partition-silent-noop` for
  the sibling silent-no-op caveat so agents don't trust the bare success on these
  CVar-driven verbs).
- **`docs/wiki-src/world_partition.md`** — note that `create_datalayer` returns
  only `DataLayer '<name>' created.` with no asset path/field echo, and name the
  live way to confirm a created data layer (e.g. re-query via the data-layer
  listing/assign verbs, or `asset.dump` on the data-layer asset) since
  `level.structure.get_level_structure_info.dataLayers` did not surface the
  newly created layer in this task.

A cheaper structural fix (out of scope for this docs-tagged ticket, but worth
noting): widen both returns to echo the observed state —
`configure_texture_streaming` to report the **read-back** `r.Streaming.PoolSize`
and whether each CVar was found (the same honesty fix
`B-configure-world-partition-silent-noop` wants), and `create_datalayer` to echo
the created instance's data-layer asset path / short name. That closes the gap at
the source; this ticket only asks for the interim documented verify guidance so
agents are unblocked now.

**Workaround:** confirm the texture pool via `system.console.search` /
`system.console.get` on `r.Streaming.PoolSize`; confirm a created data layer via
the data-layer listing/assign verbs or `asset.dump`, not via the bare success
string or `get_level_structure_info.dataLayers`.

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
