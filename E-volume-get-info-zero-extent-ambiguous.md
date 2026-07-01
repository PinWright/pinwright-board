---
id: E-volume-get-info-zero-extent-ambiguous
title: "volume.get_volumes_info reports extent {0,0,0} for empty-brush volumes with no signal that it is ground truth, not a reporting limitation — an inventory scan can't tell a broken/un-sized volume from a listing that simply can't surface bounds"
status: OPEN
severity: Low
category: ergonomic
tags: [volume, get_volumes_info, extent, readback, discoverability, docs]
encounters: 1
lastSeen: 2026-06-23T10:10:14Z
---

# `volume.get_volumes_info`'s `extent {0,0,0}` rows are ambiguous on an inventory scan

The natural first step of any "place a volume" task — *scan the level so I don't
create a duplicate* — is an unfiltered `volume.get_volumes_info`. On a populated
level that listing routinely contains rows whose `extent` (and sometimes
`location`) reads `{0,0,0}`. The method gives the caller **no way to interpret
that zero**:

- It could be a **broken/empty brush** — the ground truth for brush volumes whose
  `Brush` UModel was never initialized (the `B-blocking-volume-no-brush-geometry`
  defect: `get_volumes_info` reports `extent {0,0,0}` because the brush really is
  empty, *not* a reporting quirk).
- It could be a volume that simply **hasn't been sized yet**.
- It could *look* like a **reporting limitation** of the broad/unfiltered listing
  — which is exactly how the attempt agent read it ("no bounds surfaced in the
  broad listing").

All three render identically as `{0,0,0}`, and nothing in the response or the
wiki distinguishes them. The agent's recovery was to re-query with name/`volumeType`
filters — and even then *some* pre-existing volumes still read `{0,0,0}`, leaving
the agent unsure whether the readback was trustworthy. That uncertainty is the
friction, even though the task ended clean.

## Why this isn't just the brush bug

The code path that builds each row is **identical** for the filtered and
unfiltered cases (`VolumeHandler.cpp:1530-1547` for `AVolume`, `:1569-1586` for
`ATriggerBase`): both compute `Location = GetActorLocation()` and
`GetActorBounds(false, Origin, BoxExtent)` the same way; the `filter`/`volumeType`
params only `continue`-skip rows, they never change how location/extent are
computed. So the agent's "broad listing shows {0,0,0} but the filtered read shows
real bounds" is **not** a code-path difference — it is that the *initial*
unfiltered scan saw only pre-existing empty-brush volumes (ground-truth
`{0,0,0}`), while the *later* filtered reads hit the agent's freshly-created
`InteriorLightmassImportance` (a brush volume whose geometry was correctly built
by the now-deployed `B-blocking-volume-no-brush-geometry` #3 fix, so it has real
bounds). The two reads disagree because they looked at **different volumes**, not
because the broad listing can't report bounds — but the method surfaces nothing
that would let a caller reach that conclusion.

So the residual gap, once the brush-init bug is fixed, is purely
**interpretive/discoverability**: a `{0,0,0}` row on a scan still doesn't tell you
"this volume's brush is empty" vs "this is fine, just unsized." A field or a doc
note is needed to disambiguate.

## What it should do

- **Method (optional, preferred long-term):** surface a signal that distinguishes
  an empty brush from a legitimately-sized one — e.g. a `brushValid: false` /
  `hasGeometry: false` flag on brush-volume rows when `Brush == nullptr` (or
  `Brush->Polys->Element.Num() == 0`), so a `{0,0,0}` extent that means "broken"
  is self-describing rather than indistinguishable from "not yet sized." This
  composes with the `fields`/projection work `E-volume-get-info-no-limit-spills`
  already wants and the property-block work `E-volume-get-info-omits-physics-properties`
  wants.
- **Docs (`docs/wiki-src/volume.md`):** in the `### volume.get_volumes_info`
  section several volume tickets already target, add a note that `extent`/`location`
  come straight from `GetActorBounds()`/`GetActorLocation()`, so a volume with an
  empty/uninitialized brush legitimately reports `extent {0,0,0}` (this is ground
  truth, not a listing limitation), and that the filtered and unfiltered listings
  compute bounds identically — i.e. a `{0,0,0}` on a broad scan is the volume's
  real state, not an artifact of not passing a filter. Point at
  `actor.get_bounding_box` / `actor.describe` as the cross-check.

## Evidence

From the interior-lighting Lightmass-importance-volume struggle audit (focus
`volume.create_lightmass_importance_volume`, namespace `volume`, 11 calls,
outcome clean; the judge filed nothing for this clean task). The story explicitly
required the dedup scan first ("First check what volumes already exist in the
level so I don't create a duplicate"). The opening unfiltered
`volume.get_volumes_info` returned 16 volumes; the agent then created
`InteriorLightmassImportance` (a brush `ALightmassImportanceVolume`, spawned via
`SpawnVolumeActor`→`CreateBoxBrushForVolume`, `VolumeHandler.cpp:1168`), set its
extent then bounds, and the *filtered* readbacks returned correct live bounds
(`extent {3000,2500,1500}`, center `{-200,-200,1400}`). Friction note, verbatim:
*"the very first unfiltered get_volumes_info reports all volumes as location/extent
{0,0,0} (no bounds surfaced in the broad listing, and pre-existing
TestLightmassVolumes keep reporting zero extent even when filtered), but the
name/volumeType-filtered reads returned correct live bounds for my volume."* No
call errored — pure interpretive overhead: the agent could not tell, from the
scan, whether the `{0,0,0}` rows were broken volumes, unsized volumes, or a
listing that doesn't surface bounds, and had to re-query with filters (and still
saw `{0,0,0}` on the pre-existing TestLightmassVolumes) before trusting the
readback.

## Distinct from

- `B-blocking-volume-no-brush-geometry` (IN-REVIEW, judge's prior filing on the
  *brush* defect) — that fixes the empty brush so newly-created volumes get real
  geometry/bounds (and explains the `{0,0,0}` *is* ground truth). This ticket is
  the residual **discoverability** gap that survives the fix: even with correct
  brushes, a `{0,0,0}` extent on a scan of pre-existing/empty-brush volumes is
  uninterpretable without a `brushValid` flag or a wiki note. Not a re-file of the
  bug — a distinct process/readback angle.
- `E-volume-get-info-no-limit-spills` (OPEN) — size/Read-tax (no `limit`/projection)
  on the same verb.
- `E-volume-get-info-omits-physics-properties` (OPEN) — the verb omits
  `set_volume_properties` outputs (physics/pain/audio). Different field-coverage gap.
- `E-volume-type-filter-discovery` (OPEN) — the `volumeType` filter *value* vocabulary
  is undiscoverable. Different (request-side) gap.
- `E-volume-set-extent-units-class-dependent-docs` (OPEN) — request-side units of
  `set_volume_extent`. Different verb.

  All five (and this one) share the single `### volume.get_volumes_info` /
  `docs/wiki-src/volume.md` overlay target.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the interior-lighting Lightmass-importance-volume task (focus `volume.create_lightmass_importance_volume`, namespace `volume`, 11 calls, outcome clean; judge filed nothing). Distinct PROCESS/discoverability angle: the story's required dedup scan (unfiltered `volume.get_volumes_info`, 16 volumes) returned rows reading `location/extent {0,0,0}`, and the method gives no way to tell a ground-truth empty/broken brush (the `B-blocking-volume-no-brush-geometry` defect — `{0,0,0}` *is* the volume's real state) from an unsized volume or from a perceived "broad listing can't surface bounds" limitation. Source-confirmed the filtered and unfiltered paths compute location/extent identically (`VolumeHandler.cpp:1530-1547`/`:1569-1586` — filter only `continue`-skips rows), so the agent's "broad shows {0,0,0}, filtered shows real bounds" is the two reads hitting *different* volumes (pre-existing empty-brush vs the agent's freshly-built `InteriorLightmassImportance`), not a reporting difference — yet nothing surfaces that. Friction note verbatim: *"the very first unfiltered get_volumes_info reports all volumes as location/extent {0,0,0} (no bounds surfaced in the broad listing, and pre-existing TestLightmassVolumes keep reporting zero extent even when filtered), but the name/volumeType-filtered reads returned correct live bounds for my volume."* Proposed: a `brushValid`/`hasGeometry` flag on brush-volume rows (so a `{0,0,0}` that means "empty brush" is self-describing), plus a `### volume.get_volumes_info` note in `docs/wiki-src/volume.md` that `extent`/`location` come from `GetActorBounds()`/`GetActorLocation()` (an empty brush legitimately reports `{0,0,0}` — ground truth, not a listing limitation — and filtered vs unfiltered compute bounds identically), pointing at `actor.get_bounding_box`/`actor.describe` for cross-check. Distinct from `B-blocking-volume-no-brush-geometry` (the brush fix — this is the residual interpretive gap), `E-volume-get-info-no-limit-spills` (size/Read-tax), `E-volume-get-info-omits-physics-properties` (physics-property coverage), `E-volume-type-filter-discovery` (filter-value vocabulary), and `E-volume-set-extent-units-class-dependent-docs` (request-side units) — all sharing the one `### volume.get_volumes_info` wiki overlay.
</content>
</invoke>
