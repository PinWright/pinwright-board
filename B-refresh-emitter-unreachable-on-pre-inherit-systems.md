---
id: B-refresh-emitter-unreachable-on-pre-inherit-systems
title: "niagara.refresh_emitter cannot reach any system authored before the inherit fix — every existing handle is parent:{inherited:false}, no verb can attach a parent, and stock-template emitters refuse inherit:true, so the destructive re-add remains the only route for all content already on disk"
status: OPEN
severity: High
category: bug
tags: [niagara, refresh-emitter, add-emitter, inherit, emitter-parent, migration, pre-existing-content, snapshot, destructive-recovery, bisinheritable]
encounters: 1
lastSeen: 2026-09-05
---

# `refresh_emitter` is unreachable for every system already on disk

`niagara.refresh_emitter` is the fix for `B-niagara-add-emitter-snapshots-emitter-silently`: it
merges a parent emitter asset's changes into the system handles that inherit from it **while
keeping the system's own overrides**, which `remove_emitter` + `add_emitter` destroys. It works —
verified independently on a freshly built system (`mergesApplied: 1`, parent's new module arrived,
system-side override survived).

**It cannot be applied to any content authored before the fix**, which on this project is all of it.

## Measurement

`NS_Impact_Water` (`/Game/FPS/VFX/`), five handles — Droplets, Column, Crown, Mist, Ring.
`niagara.inspect {includeProperties:true}` reports

```
versionedEmitterData.parent = {"inherited": false}
```

on **all five**, in both the `emitters[]` and the `system.emitterHandles[]` projections. The system
was assembled with the pre-rebuild `add_emitter`, which had no `inherit` parameter and always took
the unlinked snapshot. Per `refresh_emitter`'s own contract a handle with nothing inherited is
`EMITTER_NOT_INHERITED` — an error, not a zero-item success.

## Why there is no migration

Three obstacles, and they compound:

1. **Nothing can attach a parent to an existing handle.** There is no inverse of the Niagara
   editor's *Remove Parent Emitter*, so an unlinked handle cannot be re-linked in place.
2. **The only route back is the destructive re-add** — `remove_emitter` + `add_emitter {inherit:true}`
   — which discards every edit made to the system's own copy of that emitter. That is precisely the
   data loss `B-niagara-add-emitter-snapshots-emitter-silently` documents, so the migration path to
   the fix runs through the defect the fix exists to remove.
3. **Stock-template emitters refuse to be parents.** These emitters were duplicated from the engine
   templates under `/Niagara/DefaultAssets/Templates/Emitters/`, which ship `bIsInheritable=false`
   and reject `inherit:true` with `EMITTER_NOT_INHERITABLE`. Each asset needs a `property.set` on
   `bIsInheritable` before it can be a parent at all.

## Cost

On this project the affected set is every Niagara system: 13 shipping systems across the FPS VFX
stream, plus their emitters. The practical consequence is that the workaround this ticket's sibling
documents — edit the **system's** copy, never the emitter asset, then hand-mirror the values back —
remains mandatory for all existing content, and the two copies can silently diverge in between. A
full asset-vs-system diff keyed on `(scriptUsage, name, index)` is currently the only way to prove
they agree; that is what was used to reconcile `NS_Impact_Water` after its system copies were
edited in place while its emitter assets went stale.

## Second, smaller defect found in the same verb

With the default `compile:false`, `refresh_emitter` **returns an error** —
`NIAGARA_DATA_INTERFACE_MISMATCH` — while its own payload simultaneously reports `mergesApplied: 1`.
The merge has already been applied in memory; only the compile is missing. A caller reading the
error and stopping will believe nothing happened when something did. Passing `compile:true` avoids
it, and the page already advises that for a different reason (nothing reaches a running effect
until the system is recompiled and saved), but the error-beside-success payload is misleading on
its own terms.

## Expected

Either a verb that attaches a parent emitter to an existing unlinked handle (making migration
non-destructive), or `refresh_emitter` documenting plainly that it applies only to systems authored
after the `inherit` fix, so callers with existing content do not plan around a capability they
cannot use. The `mergesApplied: 1` beside `NIAGARA_DATA_INTERFACE_MISMATCH` should not be an error
response.

## History

- `#1-filed` `OPEN` VFX — Found while reconciling `NS_Impact_Water`, whose system copies held the only correct version of a set of edits (Droplets cone axis and velocity, Crown shape primitive) while its standalone emitter assets had gone stale. `refresh_emitter` looked like the safe reconcile and was measured to be unavailable before it was called; the reconcile was done by forward-applying the values onto the emitter assets by hand and proving equality with a full module/input/static-switch diff (0 differences). The `inherited: false` measurement and the `bIsInheritable=false` template obstacle were established by one agent; the `NIAGARA_DATA_INTERFACE_MISMATCH`-beside-`mergesApplied:1` behaviour and the working end-to-end verification on a fresh system were established independently by a second agent on a scratch asset. Both were re-derived against this checkout before filing.
