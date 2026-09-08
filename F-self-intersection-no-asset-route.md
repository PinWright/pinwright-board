---
id: F-self-intersection-no-asset-route
title: "Self-intersection is measurable on a DynamicMeshActor and on .pwmodel source, and on nothing else — geometry.check_health requires actorName and geometry.audit_static_meshes has no self_intersection check, so an asset-only agent cannot evaluate the third term of the published health gate on any shipped StaticMesh"
status: OPEN
severity: Medium
category: feature
tags: [geometry, check_health, audit_static_meshes, self-intersection, static-mesh, asset-only, world-lock, health-gate, multi-agent, weapons]
encounters: 1
costly: 0
lastSeen: 2026-09-08
---

# The one health term that cannot be asked about a shipped asset

The plugin measures self-intersection well — `GeometryUtils::MeasureMeshSelfIntersection`, one AABB
tree per connected component, coplanar and transversal cases together, declined-vs-clean stated
rather than implied. It is reachable from exactly two surfaces, and neither of them takes an asset
path:

| surface | what it needs | what it answers about |
|---|---|---|
| `geometry.check_health` | **`actorName`** — a `DynamicMeshActor` that must exist in the active level | that actor's dynamic mesh |
| `model.validate` / `model.compile` | `text` or `filePath` — a `.pwmodel` document | the **source**, recompiled |

There is no third row. The verb that *does* sweep saved `UStaticMesh` assets — `geometry.audit_static_meshes`,
explicitly spawn-free ("Nothing is spawned — unlike `geometry.create_from_static_mesh`, which reaches
a saved mesh only by placing an actor for it, one per asset") — publishes ten checks and none of them
is self-intersection: `inverted`, `inconsistent_winding`, `empty`, `not_closed`, `degenerate_triangles`,
`non_manifold`, `mirrored_build_scale`, `z_fighting`, `floating_components`, `thin_shell`.

## Why the two existing surfaces are not the answer

**`geometry.check_health` needs the world.** Reaching a saved mesh from it means
`geometry.create_from_static_mesh`, which by its own page loads the asset "into an editable
DynamicMeshActor" — an actor spawned into the active level. That is the world lock. On this project
asset-only agents do not take it by protocol, and on a shared editor only one agent holds it at a
time, so the measurement is unavailable to every other agent in the session and is serialised for the
one that has it. It also mutates the level to answer a read-only question, and leaves an actor to
clean up.

**`model.validate` answers about the source, not the artifact.** It takes `.pwmodel` text or a file
path, so it cannot be pointed at a `.uasset` at all, and it only exists for meshes that *have* a
`.pwmodel` — an imported or converted StaticMesh has no source to validate. Its own page says it
"runs real engine ops on real meshes; only asset creation is skipped": it is a **full compile minus
the write**, minutes of engine work, not a read. Using it as a health probe on a shipped asset also
assumes source and asset have not diverged, which is exactly the assumption a verification step is
supposed to test.

## Why this matters more than a missing convenience

Self-intersection is not an optional extra field — it is **the third term of the gate the docs
publish**. `docs/pwmodel-format.md` and the `geometry.check_health` page both state the gate as
`isClosed && signedVolume > 0 && selfIntersections === 0`, and the `check_health` page is explicit
about why: *"Nothing above can see a surface that passes through **itself**, and both shapes of that
fault walk straight through `isClosed && signedVolume > 0`."* `B-pwmodel-health-blind-to-interior-membrane`
(DONE) and `B-pwmodel-health-no-self-intersection` (DONE) exist precisely because the first two terms
return green on meshes that are not solids.

So the shipped artifact — the thing that goes in the level, the thing a review looks at — is the one
representation on which that term cannot be evaluated at all. `geometry.audit_static_meshes` answers
the first two terms over a whole folder in one call, without a lock, and stops one term short of the
gate its sibling documents.

**Measured consequence, WEAPONS critic review round 6, 2026-09-08:** `SM_WPN_AR` self-intersection
has been **unverified for six consecutive review rounds**. The audit sweep reports it clean on
`inverted`, `not_closed`, `degenerate_triangles` and `non_manifold` every round; the one term that
could distinguish a closed solid from a closed surface pushed through itself is not in the sweep, and
the review has recorded it unmeasured each time. This is a weapon assembled from interpenetrating
`.pwmodel` parts and repeatedly re-authored (rail rebuild, barrel sliver removal, 22,652 -> 11,110 ->
11,630 triangles) — the content class most likely to acquire the fault, checked with the one
instrument that cannot see it.

## What is asked for

Either, in preference order:

1. **A `self_intersection` check id in `geometry.audit_static_meshes`.** The measurement already
   exists in `GeometryUtils` and this verb already loads and reads the mesh, already runs per-component
   analysis (`inverted` walks edge-connected components and reports each one's volume), and already
   has the vocabulary for an expensive check that may decline — `unrunnable`, per-check `flagged /
   clean / unrunnable / not-applicable`. The existing budget behaviour transfers unchanged: declined
   above 200k triangles, pair count capped, `selfIntersectionsTruncated` reported. Make it opt-in
   rather than default if the cost class is the concern — `thin_shell` is already precedent for a
   check that is off unless asked for.
2. **An `assetPath` form of `geometry.check_health`**, reading the saved mesh the way
   `audit_static_meshes` does rather than by spawning. This also closes the more general gap that
   every field `check_health` publishes is actor-only.

Either one gives an asset-only agent a lock-free route to the third gate term. Neither needs new
geometry code.

## Distinctness (dedup)

Searched before filing — `check_health`, `self-intersect` / `selfIntersect`, `DynamicMeshActor`,
`audit_static_meshes` and `world lock` across all ~1875 tickets:

- `E-geometry-check-health-blind-to-self-intersection` (**IN-REVIEW**) — the closest, and **already
  satisfied**: it asked for field-set *parity* between `check_health` and the `model.*` health block,
  and the developer entry records `checkSelfIntersection` shipped with `selfIntersectionsMeasured`.
  That fix left `actorName` required, which is this ticket: parity of *fields* on a surface that
  still cannot be pointed at an asset. Appending here would reopen a ticket whose ask was met.
- `B-pwmodel-health-no-self-intersection` (DONE) / `B-pwmodel-health-blind-to-interior-membrane`
  (DONE) — added the measurement to the `model.*` **source** surface. This ticket is about the
  artifact surface, which those two deliberately scoped out.
- `B-pwmodel-overlap-check-skips-modifiers` (OPEN) — overlap **between parts**, a `.pwmodel`-compile
  diagnostic; different fault, different surface.
- `F-mesh-capture-without-asset-editor-or-world-lock` (IN-REVIEW, High) — **the same structural
  complaint one instrument over**: no static-mesh *capture* route that avoids the asset editor and
  the world lock. Cross-reference, do not merge — that ticket asks for pixels, this one asks for a
  number, and a fixer could ship either without the other. Two tickets now describing the same shape
  is itself a signal about where the asset-only surface is thin.
- `F-static-mesh-uv-channel-readout` (IN-REVIEW) — third instance of the shape (UV layout
  "unverifiable without spawning a DynamicMeshActor"), different measurement.
- `B-mesh-audit-includeclean-emits-nothing`, `B-mesh-audit-package-path-reads-as-broken-asset`,
  `B-mesh-audit-zfight-coarse-grid-unrunnable-on-tiny-meshes` — same verb, all three about checks
  that exist; this is about a check that does not.

## Severity

**Medium.** Impact class: a hard blocker for an asset-only caller, but a **soft** one overall —
the answer is reachable by taking the world lock and spending `create_from_static_mesh` +
`check_health` + cleanup per asset, which is a documented workaround at many extra calls, and for
`.pwmodel`-sourced meshes `model.validate` gets close (at full-recompile cost, and about the source
rather than the artifact). No wrong or stale data is returned: `check_health` states
`selfIntersectionsMeasured`, and the audit reports its check set honestly, so nothing here reads as a
clean answer that was not measured. Reach: no bump — `audit_static_meshes` runs in every content
review round, which is normal rather than every-session. A fixer who rates it **High** on the
grounds that the third term of a documented gate is unavailable on the shipped artifact, with no
lock-free route of any kind, will get no argument from this reporter.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Filed in WEAPONS critic review round 6, 2026-09-08, against this checkout's generated wiki (no plugin source opened, no `file:line` claimed). Verified from the wiki at `X:/src/unreal/EAContentExamples58/Saved/PinWright/wiki/`: `geometry.check_health.md` lists `actorName` (**required**, "Name of the DynamicMeshActor") and `checkSelfIntersection` (optional) and nothing else — no asset path in any form; `geometry.create_from_static_mesh.md` reaches a saved mesh only by spawning a `DynamicMeshActor`, i.e. through the world lock; `geometry.audit_static_meshes.md` is the spawn-free asset sweep and its published check set is `inverted`, `inconsistent_winding`, `empty`, `not_closed`, `degenerate_triangles`, `non_manifold`, `mirrored_build_scale`, `z_fighting`, `floating_components`, `thin_shell` — **no self-intersection check**; `model.validate.md` takes `text` or `filePath` (`.pwmodel` source only) and "runs real engine ops on real meshes", a full compile minus the asset write rather than a read. Net: self-intersection is measurable on a level actor or on a source document and on no shipped asset, so an agent that by protocol does not take the world lock cannot evaluate `selfIntersections === 0` — the third term of the gate `docs/pwmodel-format.md` and the `check_health` page both publish — on anything that actually ships. Measured consequence: `SM_WPN_AR` self-intersection unverified for **six consecutive review rounds**, while the audit sweep returns it clean on the four topology checks it does run; it is a weapon assembled from interpenetrating `.pwmodel` parts and repeatedly re-authored, i.e. the content most likely to acquire the fault. Asked for, cheapest first: a `self_intersection` check id in `geometry.audit_static_meshes` (the measurement exists as `GeometryUtils::MeasureMeshSelfIntersection`, the verb already loads the mesh and already walks connected components for `inverted`, and `unrunnable` / `thin_shell` are existing precedent for an expensive opt-in check that may decline), or an `assetPath` form of `geometry.check_health`. Deduped board-wide before filing: `E-geometry-check-health-blind-to-self-intersection` (IN-REVIEW) is the nearest and is **already satisfied** — it asked for field parity, which `checkSelfIntersection` delivered while leaving `actorName` required, so appending there would reopen a met ask; the two `B-pwmodel-health-*` tickets (DONE) added the measurement to the source surface and scoped the artifact surface out; `F-mesh-capture-without-asset-editor-or-world-lock` and `F-static-mesh-uv-channel-readout` are the same structural shape on different instruments (pixels, UV layout) and are cross-referenced rather than merged. `encounters: 1`, `costly: 0` — this encounter recorded an absent answer, not expended work, which the cost modifier calls cheap.
