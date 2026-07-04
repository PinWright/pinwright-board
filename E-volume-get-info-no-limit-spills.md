---
id: E-volume-get-info-no-limit-spills
title: "volume.get_volumes_info has no limit/projection — full level dump (engine volumes) overflows the 10k spill threshold and forces a Read of the HttpResponses file"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [volume, response-size, oversized, pagination, docs]
encounters: 4
lastSeen: 2026-07-02T00:51:24.6212753+03:00
---

# volume.get_volumes_info has no limit/projection — it dumps every volume and spills to file

`volume.get_volumes_info` enumerates **every** `AVolume` and `ATriggerBase` in the
editor world and returns the full array in one payload, with only two optional
*substring* narrowing params (`filter` = name/label, `volumeType` = class) and **no
`limit`, no pagination cursor, and no field projection** (VolumeHandler.cpp:1536-1648).
On any populated level the engine's own bookkeeping volumes — many duplicate
`PostProcessVolume` / `LightmassImportanceVolume` / etc. instances — dominate the
result. The friction task saw `totalCount 19` (then `17` after two removals), and the
serialized array crossed the **10000-char spill threshold**, so the response came back
as `outputTooLong` and the full payload was written to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing the agent to
**Read/Grep the spilled file** just to read back the names of the three volumes it had
just created.

The agent's actual intent on both readback steps was tiny — step 5 "confirm my 3 new
volumes are present and read back their names", step 9 "verify only `Arena_BossTrigger`
remains of my set". A handful of named rows. But the method has no way to ask for "just
these / just the names / just the first N", so the common verify-my-own-work case always
pays the full-level dump + spill-to-file + extra Read tax.

## What's wrong

`VolumeHandler.cpp:1536` registers only `filter` and `volumeType`. The body loops
`TActorIterator<AVolume>` (:1564) and `TActorIterator<ATriggerBase>` (:1603),
unconditionally appending one object per volume (name, class, location, full
`extent`) with no cap and no `fields`/`limit` gate, then emits the whole
`volumesInfo.volumes` array (:1642-1648). The substring filters help only if the caller
already knows a discriminating name/type — and `volumeType`'s accepted vocabulary is
itself undiscoverable (see `E-volume-type-filter-discovery`), so the natural "just show
me everything" call is the one that overflows.

## What it should do

Mirror the **actor.list** family fix already shipped in `QueryHandler.cpp` (the
`E-actor-list-no-limit-spills` template) — NOT a small truncating default:

- Add an optional `limit` with **default `0` = all** (byte-identical default output, so it
  never silently elides rows). It caps only when the caller explicitly sets it; a
  `totalCount` keeps reporting the untruncated match count and a `truncated` flag marks
  when the array was capped, so elision is always detectable. A *small* default was
  rejected: the engine's PostProcess/Lightmass volumes iterate first and the caller's own
  just-created volumes tend to land at the tail of `TActorIterator` order, so a small
  default would drop exactly the rows a verify-my-own-work readback wants. The `limit` is
  the opt-in "just peek at N" control; it is NOT what keeps the common readback inline.
- The **`namesOnly` / `fields` projection is the real payload reducer**: it drops the
  verbose per-row `location`+`extent` (the bulk of each row's bytes), so the common "read
  back the labels I created" call returns just `name`(+`class`) and stays well under the
  spill threshold. `namesOnly=true` keeps name+class; `fields=[...]` is an explicit
  allow-list over `name`/`class`/`location`/`extent` (a bare string is the single-key
  shorthand). Reuses the shared `FHandlerContext::ReadFieldProjection` helper.
- Discoverability rides on the handler **summary string** (as actor.list's does): it now
  enumerates the `filter`/`volumeType`/`limit`/`namesOnly`/`fields` narrowing options and
  states that the reader returns *all* volumes including engine-internal ones so a
  populated level can spill. The standalone `docs/wiki-src/volume.md` overlay note is left
  to the sibling tickets that already own that overlay
  (`E-volume-type-filter-discovery`, `E-volume-get-info-omits-physics-properties`) to
  avoid a cross-ticket edit collision.

## Evidence

From the arena gameplay-triggers task (focus `volume.remove_volume`, namespace
`volume`, outcome `tool_bug` for the *separate* brush-geometry bug). Friction note,
verbatim: *"later get_volumes_info calls hit the \"outputTooLong\" >10000-char threshold
and wrote the full payload to a Saved/HttpResponses JSON file, forcing a Read/Grep to
inspect results"* and *"(and lists the engine's many duplicate PostProcess/Lightmass
volumes)"*. Call-log: **four** `volume.get_volumes_info` calls in a 10-call task (initial
snapshot, a wiki-nav confirm, a post-create confirm, a final verify); the self-report
notes the post-create and final-verify calls were the ones that *"output to file"* and
required a follow-up Read/Grep — `totalCount` 19 then 17, the great majority of which are
engine volumes the agent never asked about.

## Distinct from

- `E-http-response-spill` (DONE) — that is the *generic* server-side spill mechanism
  (the file-reference fallback itself); this ticket is that a *specific verbose reader*
  has no narrowing to stay under the threshold in the first place, the same relationship
  `E-recorder-list-sessions-limit` and `E-graph-connections-pagination` have to the spill
  mechanism.
- `B-blocking-volume-no-brush-geometry` (IN-REVIEW, the judge's filing for this task) —
  that is the brush being null / the `128/128/128`-or-`0` extent being wrong *ground
  truth*; this is purely the *response size / Read-tax* on the readback path, independent
  of whether the extents are correct.
- `E-volume-type-filter-discovery` (OPEN) — that is the *filter-value vocabulary* being
  undiscoverable; this is the *absence of a size cap / projection* so even a correct call
  overflows. They share the same wiki overlay target but are different gaps.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the arena gameplay-triggers struggle audit (focus `volume.remove_volume`, outcome tool_bug). `volume.get_volumes_info` has no `limit`/pagination/projection (VolumeHandler.cpp:1486 registers only `filter`/`volumeType`; loops all `AVolume`+`ATriggerBase` :1514/:1553 with no cap), so a populated level full of engine PostProcess/Lightmass volumes (`totalCount` 19→17) pushes the payload past the 10000-char spill threshold; two of the task's four `get_volumes_info` calls returned `outputTooLong` and spilled to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing a Read/Grep just to read back three created volume names. Proposed: add `limit` (default-inline, 0=all, with untruncated `totalCount`) per `E-recorder-list-sessions-limit`/`E-graph-connections-pagination`, optionally a `fields`/`namesOnly` projection, and document the all-volumes-including-engine behavior in the `### volume.get_volumes_info` section of `docs/wiki-src/volume.md`. Distinct from `E-http-response-spill` (DONE, the spill mechanism), `B-blocking-volume-no-brush-geometry` (the extent ground-truth bug), and `E-volume-type-filter-discovery` (the filter-value vocabulary gap).
- `#2-more-evidence-pool-water-volume` `OPEN` reporter — Recurrence from the swimming-pool water-physics-volume struggle audit (focus `volume.create_physics_volume`, namespace `volume`, outcome clean). Same friction on the final readback step: the task created one `PoolWaterVolume` (`totalCount` 16→17) and called `volume.get_volumes_info` twice (initial scan inline-OK at 16 volumes; final verify at 17 crossed the 10000-char inline limit and was "auto-written to a HttpResponses JSON file that I read off disk"). Friction note verbatim: *"the final get_volumes_info exceeding the 10000-char inline display limit, so it was auto-written to a HttpResponses JSON file that I read off disk (expected paging behavior, not a blocker)"*. Confirms the no-`limit`/no-projection readback tax recurs across tasks even on a clean outcome — even a single-volume verify pays the full-level dump + spill-to-file + extra Read because the engine's PostProcess/Lightmass volumes dominate the payload. No new angle; reinforces the proposed `limit`/`namesOnly` projection.
- `#3-more-evidence-arena-killz-moat` `OPEN` reporter — Third recurrence, hazard-layer arena blockout struggle audit (focus `volume.create_kill_z_volume`, namespace `volume`, outcome clean). The task created three volumes (two `KillZVolume` + one `Moat_Water` PhysicsVolume), pushing `totalCount` 16→19, and called `volume.get_volumes_info` twice (no filter): the initial-state scan and the final readback. The *final* `get_volumes_info` (19 volumes, no filter) crossed the 10000-char inline threshold and spilled to a HttpResponses JSON file the agent had to Read separately. Friction note, verbatim: *"the final get_volumes_info exceeded the 10000-char display threshold and was spilled to a HttpResponses JSON file I had to Read separately - expected for a 19-volume scene but adds a step."* Same no-`limit`/no-projection readback tax: a 3-row verify-my-own-work intent paid the full 19-volume dump (mostly engine PostProcess/Lightmass) + spill-to-file + extra Read, because the story explicitly asked for an unfiltered final readback ("call volume.get_volumes_info again (no filter) to confirm all three new volumes"). Reinforces the proposed `limit` (default-inline, 0=all, untruncated `totalCount`) and `fields`/`namesOnly` projection; the unfiltered "confirm everything" call is precisely the one that overflows. No new angle.
- `#4-liveness-toxic-swamp` `OPEN` reporter — Still observed. Toxic-swamp hazard blockout (namespace `volume`, outcome ergo). The unfiltered `volume.get_volumes_info {}` over 20 volumes serialized to 12241 chars > 10000 and spilled to a `Saved/EditorAutomation/HttpResponses` JSON file, adding one extra Read to confirm the four created volumes. Pure reconfirm of the no-`limit`/no-projection readback tax — same shape as #1–#3, no new angle. (Primary friction this task was the field-coverage gap, filed to `E-volume-get-info-omits-physics-properties` #3, not the spill.)
- `#5-reword-and-implement` `IN-REVIEW` developer — Reworded the fix to the shipped **actor.list** precedent (`E-actor-list-no-limit-spills` / `QueryHandler.cpp`) and implemented it. The ticket's original "limit default small enough to stay inline" was corrected: a small default would elide the caller's own volumes (engine PostProcess/Lightmass iterate first; user-created volumes land at the tail of `TActorIterator` order), so `limit` now defaults to `0`=all (byte-identical default output) and only caps when explicitly set, with `totalCount` (untruncated) + a new `truncated` flag; the **`namesOnly`/`fields` projection** (dropping the verbose per-row `location`+`extent` via the shared `FHandlerContext::ReadFieldProjection`) is the actual inline-readback reducer. Files: `Handlers/Volume/VolumeHandler.cpp` (`volume.get_volumes_info`: added `limit`/`fields`/`namesOnly` params, per-field projection + limit gate in both the `AVolume` and `ATriggerBase` loops, emit `count`/`truncated`, expanded summary), `Tests/World/TestVolumeHandlers.cpp`. Regression test: `PinWright.volume.get_volumes_info.LimitAndProjection` — spawns two token-sharing trigger volumes via the production create handler, then asserts `limit:1`→count 1/totalCount 2/truncated true, `namesOnly`→rows keep name+class but drop location+extent, and `fields:[name]`→name only; it fails if the params are ignored. Discoverability delivered via the handler summary (as actor.list did); the standalone `volume.md` overlay note is left to the sibling overlay tickets to avoid a collision.
