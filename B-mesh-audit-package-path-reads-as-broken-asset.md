---
id: B-mesh-audit-package-path-reads-as-broken-asset
title: "geometry.audit_static_meshes given a package path instead of an object path reports MESH_AUDIT_UNLOADABLE on all seven checks and pass:false — a caller's argument-form mistake presented as a content defect in the asset"
status: IN-REVIEW
severity: Medium
category: bug
tags: [geometry, audit_static_meshes, asset-path, package-path, object-path, MESH_AUDIT_UNLOADABLE, misclassification, argument-validation]
encounters: 1
lastSeen: 2026-08-22T00:00:00Z
---

# A wrong argument form comes back looking like a broken asset

`geometry.audit_static_meshes` accepts an explicit `assets` array. Each entry is resolved with

```cpp
FAssetData Data = AssetRegistry.GetAssetByObjectPath(FSoftObjectPath(Sanitized));
if (Data.IsValid()) { Matched.Add(...); } else { Unresolved.Add(Entry); }
```

(`Source/PinWrightGeometry/Private/Handlers/Geometry/MeshAuditHandler.cpp:297-305`), so only the
**object-path** form resolves — `/Game/Dota2/Meshes/Trees/SM_X.SM_X`. The **package** form,
`/Game/Dota2/Meshes/Trees/SM_X`, carries no object name, produces an invalid `FSoftObjectPath`,
and lands in `Unresolved`.

What happens next is the defect. Unresolved entries are folded back into the report **as unmeasured
assets** (`:375-388`): each gets `UnrunnableCode = MESH_AUDIT_UNLOADABLE` and is pushed through
`EvaluateAsset`, so it occupies exactly the buckets a mesh that genuinely failed to load would. For
a caller with all seven checks selected the response carries **seven `Unrunnable` finding rows
against that asset path**, `assetsUnmeasured` incremented, and `pass: false`.

Nothing in that shape says "you passed the wrong string form". It says the asset could not be read
— which, for a verb whose entire output is verdicts about asset health, reads as *the asset is
broken*. A caller acting on it re-exports or re-authors a mesh that was fine.

**The folding itself is right and must not be reverted.** Its own comment says why: silently
shrinking the requested set is how a sweep reports a clean verdict over assets it never saw. The
bug is not that unresolved entries are reported — it is that a *caller error* and a *content
defect* are reported through the same channel, at the same prominence, under the same code.

## Not the same as the neighbouring ticket

`B-references-rejects-short-package-path` (OPEN, Medium) is the same resolution mechanism —
`GetAssetByObjectPath` on a form the caller reasonably supplied — on `asset.references` /
`asset.dependencies`. It stays separate: there the symptom is a hard `ASSET_NOT_FOUND` that stops
the call, which is unhelpful but unambiguous. Here the call *succeeds*, returns a fully-formed
audit structure, and the misdirection is the whole problem. Neither fix implies the other, but a
shared path-normalisation helper would resolve both and is the obvious way to take them together.

## Fix

Two shapes, either acceptable, differing in what they teach:

1. **Normalise the argument.** Accept the package form by deriving the object name
   (`/Game/A/B` → `/Game/A/B.B`) before resolving — the convention most asset verbs already teach.
   Cheapest for the caller and consistent with the rest of the surface.
2. **Refuse it at parse time.** Reject an entry that is not a resolvable object path with
   `INVALID_ARGUMENT`, naming the offending string and the expected form, **before** the sweep
   runs, so it never enters the findings structure at all.

What must NOT remain is the current middle: a caller error carried into content findings. If option
1 is taken, an entry that still fails to resolve after normalisation needs its own code
(`ASSET_NOT_FOUND`, not `MESH_AUDIT_UNLOADABLE`) so "no such asset" and "this mesh would not load"
stay distinguishable — different facts, and only one of them is about geometry.

Either way the wiki page must state the accepted form for `assets` explicitly.

## Severity

**Medium**, and the reasoning is recorded because the band is arguable.

The rubric's **High** band is silent wrong data the caller trusts and builds on. This is not quite
silent: `UnrunnableReason` reads `'<path>' did not resolve to a StaticMesh asset.`, which names the
real cause. A caller who reads the per-finding rows is not deceived, so High does not apply on its
own terms.

It is well above pure friction, though. The caller does not merely have to look something up — they
receive a fully-formed **false verdict** (`pass: false`) about a healthy asset, and that verdict is
precisely what the verb exists to produce. That is the rubric's **Medium** soft-blocker band:
workable, but only by reading past the summary into the rows and knowing to distrust it.

**No reach modifier.** `geometry.audit_static_meshes` shipped today and is not yet an every-session
method; bumping up for reach would predict adoption rather than measure it. Revisit if the verb
enters a routine content sweep, since the misclassification cost scales with how many callers hit
it before learning the form.

Filed against a verb that shipped **today** — a fresh defect on a new surface, not a legacy one,
and the same confusion class the surrounding wave has been eliminating: a caller error wearing the
costume of a content defect, inside a green-looking structure.

## History
- `#1-package-path-reported-as-unloadable` `OPEN` reporter — `geometry.audit_static_meshes` resolves each `assets` entry with `AssetRegistry.GetAssetByObjectPath(FSoftObjectPath(Sanitized))` (`MeshAuditHandler.cpp:300`), so only the object-path form `/Game/.../SM_X.SM_X` resolves; the package form `/Game/.../SM_X` produces an invalid `FSoftObjectPath` and falls to `Unresolved` (`:305`). Unresolved entries are then folded back into the report as unmeasured assets with `UnrunnableCode = MESH_AUDIT_UNLOADABLE` and pushed through `EvaluateAsset` (`:375-388`), so they occupy the same buckets a mesh that failed to load would: seven `Unrunnable` finding rows with all checks selected, `assetsUnmeasured` incremented, `pass: false`. The result is a caller's argument-form mistake presented as a content defect in the asset — a caller acting on it re-authors a mesh that was fine. The folding is correct in itself and must not be reverted; its own comment gives the reason (silently shrinking the requested set is how a sweep reports a clean verdict over assets it never saw). The defect is that a caller error and a content defect travel the same channel at the same prominence under the same code. Fix either by normalising `/Game/A/B` to `/Game/A/B.B` before resolving, or by refusing a non-resolvable entry with `INVALID_ARGUMENT` before the sweep runs; if normalising, a still-unresolved entry needs `ASSET_NOT_FOUND` rather than `MESH_AUDIT_UNLOADABLE` so "no such asset" and "this mesh would not load" stay distinguishable. Deduped against the board: `B-references-rejects-short-package-path` (OPEN, Medium) is the same resolution mechanism on `asset.references` / `asset.dependencies`, but its symptom is an unambiguous hard `ASSET_NOT_FOUND` that stops the call, where this one succeeds and returns a false verdict — separate tickets, though one shared path-normalisation helper would fix both. Severity **Medium**, argued rather than asserted: the rubric's High band needs silent wrong data, and `UnrunnableReason` does name the real cause (`'<path>' did not resolve to a StaticMesh asset.`), so a caller who reads the rows is not deceived; but it is well above friction, since the summary verdict `pass: false` about a healthy asset is exactly what the verb exists to produce — the Medium soft-blocker band. No reach bump: the verb shipped today, so bumping for reach would predict adoption rather than measure it. Revisit if it enters a routine content sweep.
- `#2-normalized-with-a-distinct-absent-code` `IN-REVIEW` fixer — Fixed by **option 1 (normalise)**, chosen over option 2 (refuse) because the package form is what the rest of the asset surface teaches and refusing it here would make this verb the odd one out. `PathUtils::NormalizeToObjectPath` (`Source/PinWright/Private/Utils/PathUtils.{h,cpp}`) maps `/Game/A/SM_X` -> `/Game/A/SM_X.SM_X`, leaves an object or subobject path untouched, and is idempotent. `MeshAuditHandler.cpp` now splits the three facts the ticket says must stay apart: a string that is not a content path at all is rejected with `INVALID_ARGUMENT` **before the sweep runs**, naming both accepted forms, so it never enters the findings structure; a well-formed path the registry has nothing at is folded in as before but coded `ERR_ASSET_NOT_FOUND`, not `ERR_MESH_AUDIT_UNLOADABLE`, with a reason saying the geometry was not measured; a well-formed path that resolves is audited. The folding itself is untouched, per the ticket. Tests: `PinWright.core.path.normalize_to_object_path.PackageFormBecomesObjectPath` / `.MalformedInputIsRejectedWithAReason` (ten malformed forms, each asserted to fail with a reason and to leave the out-param empty) and `PinWright.Geometry.MeshAudit.MalformedAssetPathIsRejectedNotReportedAsABrokenMesh` (three malformed shapes, each asserted to come back as an RPC error whose code is `INVALID_ARGUMENT` and explicitly not a `MESH_AUDIT_*` code). Docs: `Docs/wiki-src/geometry.md` states both accepted forms and the three outcomes; `Docs/rpc-design.md` §18 records the argument-error-is-not-a-finding boundary with this ticket as the worked example. **Not verified end to end by an automated test:** the absent-asset leg's `ASSET_NOT_FOUND` code travels in the job result, and this verb returns a job ticket, for which the suite has no established driving pattern — that leg is covered by inspection only. **Sibling `B-references-rejects-short-package-path` is deliberately NOT touched**, but `NormalizeToObjectPath` is the shared helper that ticket names, placed in `PathUtils` rather than in the geometry module so adopting it there is a small change.
