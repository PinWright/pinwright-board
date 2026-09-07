---
id: B-mesh-audit-includeclean-emits-nothing
title: "geometry.audit_static_meshes includeClean:true emits nothing on any sweep that produced findings — cleanAssets is suppressed when empty, and an asset only qualifies as clean if it drew zero findings across all ten checks"
status: OPEN
severity: Medium
category: bug
tags: [geometry, audit_static_meshes, includeClean, cleanAssets, silent-no-op, measurements, readback, weapons]
encounters: 1
costly: 1
lastSeen: 2026-09-06T00:00:00Z
---

# A flag that reads as ignored, and a measurement channel that only opens for failures

`geometry.audit_static_meshes {includeClean: true}` on a sweep that produced findings comes back
with **no `clean` key, no `cleanAssets` key, nothing**. From the caller's side the flag is
indistinguishable from a parameter that was accepted and dropped.

## What came back — measured

Called with `includeClean: true` on a weapon-mesh sweep that produced findings. The response
carries `findings[]`, the per-check tallies, `assetsMatched` / `assetsFlagged` / `assetsUnmeasured`
— and no key naming the clean assets at any level. Re-reading the response for a differently-spelled
key found none.

## Root cause — confirmed in source

Two independent mechanisms stack, and either alone would produce the observed nothing.

**1. The output key is suppressed when the array is empty.**

```cpp
if (Report.CleanAssets.Num() > 0)
{
    ...
    Result->SetArrayField(TEXT("cleanAssets"), CleanRows);
}
```
`Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/MeshAuditHandler.cpp:542-551`

So an empty clean set is not reported as an empty array — the key vanishes. The caller cannot tell
"the flag did nothing" from "the flag worked and the answer is none", which is the whole reason the
flag looks like a no-op.

**2. "Clean" means clean across every selected check, per asset.**

```cpp
if (R.ErrorCount != ErrorsBefore || R.WarningCount != WarningsBefore
    || R.UnrunnableCount != UnrunnableBefore)
{
    ++R.AssetsFlagged;
}
else if (R.CleanAssets.Num() < Config.MaxCleanAssets)
{
    R.CleanAssets.Add(AssetPath);
}
```
`Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/MeshAuditUtils.cpp:2947-2955`

An asset enters `CleanAssets` only if it produced **zero** errors, warnings *and* unrunnables across
the whole sweep. One `warning`-severity finding on one of ten checks disqualifies it. On a sweep of
a real kit that is most assets, so `CleanAssets` is routinely empty — and then mechanism 1 hides
even that fact. The plumbing is otherwise live: `includeClean` is a declared param
(`MeshAuditHandler.cpp:136-137`) and sets `Config.MaxCleanAssets = 500` (`:260`,
`MeshAuditUtils.h:516-517`).

**Naming note.** There is also a `clean` field in the response, but it is a *per-check tally count*
on each check row (`MeshAuditHandler.cpp:514`), not the `includeClean` output. A caller looking for
"the clean list" finds a number that means something else.

## The second half: a clean asset yields no measurements at all

Even when `cleanAssets` does appear, it is a flat array of **path strings**
(`MeshAuditHandler.cpp:544-549`) — the param help says so: *"Also list the asset paths that produced
no finding at all (capped at 500)."* Measurements ride on **findings**
(`Result->SetObjectField(TEXT("measurements"), ...)`, `MeshAuditHandler.cpp:534-537`), so an asset
with no finding produces no `measurements` block anywhere.

That makes the audit unusable as a measuring instrument on healthy content. To read a clean asset's
component count and signed volumes this round, the only route was to **force an UNKNOWN verdict** by
setting `minVolumeRatio: 0.99` — manufacturing a finding purely to open the measurement channel.
The threshold that decides the verdict is also the switch that decides whether numbers are emitted.

## What is asked for

1. **Always emit `cleanAssets` when `includeClean` is true**, as an empty array when there are none.
   A flag whose success case is byte-identical to being ignored teaches callers to stop trusting it.
   This is a two-line change at `MeshAuditHandler.cpp:542`.
2. **Emit per-asset measurements for clean assets**, so the verb can measure content that is not
   broken. Either promote `cleanAssets[]` from strings to rows carrying the same `measurements`
   object a finding gets, or add a separate per-asset measurement block keyed by asset path.
3. **Failing (1) and (2), document what the flag does.** The param help currently promises "list the
   asset paths that produced no finding at all" and does not say that the key is omitted when the
   list is empty, nor that a single warning on any of ten checks disqualifies an asset, nor that no
   measurements accompany a clean asset. Any one of those three, stated, would have prevented this
   report.

Note the interaction worth stating in the docs whichever way this lands: `pass` requires zero
unrunnables (see `B-mesh-audit-zfight-coarse-grid-unrunnable-on-tiny-meshes`), and the same
unrunnable count disqualifies an asset from `cleanAssets` here — so on a kit that trips that ticket,
`includeClean` is guaranteed to return nothing for a reason that has nothing to do with mesh health.

## Severity

**Medium**, the soft-blocker band. The information is obtainable — the `minVolumeRatio: 0.99`
forcing trick works and was used — but only via an undocumented abuse of a threshold parameter, and
the flag's silent-nothing shape is what sends a caller looking for that trick in the first place.
Not High: no wrong data is returned and nothing is corrupted, the response is merely missing a key.
Reach is the folder-sweep path, which is the normal way this verb is used, so no downward adjustment.

## Related

- `B-mesh-audit-package-path-reads-as-broken-asset` (IN-REVIEW) — same verb, same response builder;
  there a caller error is reported through the findings channel, here a success is reported through
  no channel at all.
- `B-mesh-audit-zfight-coarse-grid-unrunnable-on-tiny-meshes` (OPEN) — same verb; its unrunnables are
  one of the three counters that disqualify an asset from `cleanAssets`, so the two interact as
  described above.
- `F-static-mesh-section-material-map` (IN-REVIEW) — the measurement surface a caller falls back to
  when the audit will not measure clean content; its `sections[]` / `slotUsage[]` are what actually
  answered the per-asset questions this round.
- `B-actor-list-fields-unknown-key-silently-dropped` — the same failure class on a different verb: a
  flag accepted, and nothing in the response to say whether it did anything.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 4. `geometry.audit_static_meshes {includeClean: true}` on a sweep that produced findings returned no `clean` key and no `cleanAssets` key at any level, so the flag reads as a silent no-op. Root cause **confirmed in source** (not inferred), two stacked mechanisms: (a) the emit is guarded by `if (Report.CleanAssets.Num() > 0)` (`Handlers/Geometry/MeshAuditHandler.cpp:542-551`), so an empty clean set drops the key entirely rather than returning an empty array, leaving "flag ignored" and "flag worked, answer is none" indistinguishable; and (b) an asset only enters `CleanAssets` if it produced zero errors AND zero warnings AND zero unrunnables across the whole sweep (`Handlers/Geometry/MeshAuditUtils.cpp:2947-2955`), so one warning-severity finding on one of ten checks disqualifies it and the set is routinely empty on a real kit. The plumbing is otherwise live — `includeClean` is declared at `MeshAuditHandler.cpp:136-137` and sets `Config.MaxCleanAssets = 500` at `:260` (`MeshAuditUtils.h:516-517`). Naming trap worth noting: the response does carry a `clean` field, but it is a per-check tally count on each check row (`MeshAuditHandler.cpp:514`), not the `includeClean` output. Second, related gap in the same verb: `cleanAssets` is a flat array of path strings (`:544-549`, matching its param help) and measurements ride only on findings (`:534-537`), so an asset with no finding yields no measurements at all — the only way to read a clean asset's component count and signed volumes this round was to force an UNKNOWN verdict with `minVolumeRatio: 0.99`, manufacturing a finding to open the measurement channel. Ask: always emit `cleanAssets` (empty array included) when the flag is true — a two-line change at `:542`; emit per-asset measurements for clean assets, either by promoting `cleanAssets[]` to rows carrying the same `measurements` object findings get or by a separate per-asset block; or, failing both, document that the key is omitted when empty, that one warning on any of ten checks disqualifies an asset, and that clean assets carry no measurements. Dedup: ripgrep for `includeClean` across the whole board returned zero hits, and the two existing `B-mesh-audit-*` tickets own different failures on the same verb (package-path-as-broken-asset; z_fighting unrunnable on coarse meshes) — noting that the latter's unrunnables are one of the three counters that disqualify an asset here, so on a kit that trips it `includeClean` returns nothing for a reason unrelated to mesh health. Severity Medium on the soft-blocker band: obtainable via the undocumented `minVolumeRatio` forcing trick, nothing wrong or corrupted, but the silent-nothing shape is what sends a caller hunting for the trick.
