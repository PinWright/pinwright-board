---
id: E-spline-point-tangent-readback-gap
title: "spline.set_spline_point_tangents applies a custom tangent but no readback surfaces it — its own response doesn't echo the applied tangent and get_splines_info's per-point rows carry only index/location/type, so the exact tangent vector can only be inferred (point-type flip to CurveCustomTangent + splineLength change), never verified directly"
status: OPEN
severity: Low
category: ergonomic
tags: [spline, set_spline_point_tangents, get_splines_info, readback, tangent, verify-after-mutate]
encounters: 1
lastSeen: 2026-07-13T09:07:03.7514845+03:00
---

# No readback surfaces per-control-point tangent vectors after `spline.set_spline_point_tangents`

`spline.set_spline_point_tangents` sets a custom arrive/leave tangent on a spline
control point (auto-promoting that point's type to `CurveCustomTangent`), but the
applied tangent value is **write-only from the caller's perspective**: nothing in
the spline namespace lets you read it back.

- The mutation verb's **own response does not echo the applied tangent** it just
  wrote — the caller cannot confirm from the call result that the vector landed
  as intended (magnitude/direction).
- The prescribed readback `spline.get_splines_info` reports **per-point
  `index` / `location` / `type` only** — its `points[]` rows carry no
  `arriveTangent` / `leaveTangent` fields. So even a fresh independent readback
  cannot corroborate the tangent vector.

The net effect: after tuning a control point with `set_spline_point_tangents`,
the caller can only **infer** that the tangent took hold — indirectly, from (a)
the point's `type` flipping `Curve` → `CurveCustomTangent` in `get_splines_info`,
and (b) the overall `splineLength` growing (a strong sideways tangent bows the
curve out, so it gets longer). Neither confirms the actual tangent *value*; a
tangent applied in the wrong direction, or with the wrong magnitude, would still
flip the type and change the length, so the inference cannot distinguish a
correct tangent from a merely-nonzero one.

This is a **readback-completeness** gap, not a false-success: `get_splines_info`
does not *lie* about tangents, it simply omits them, and the honest point-type +
length signals let the common "did my custom tangent take?" question be answered
by inference. It is distinct from every existing spline-readback ticket, and no
sibling verb fills the gap:

- `E-get-splines-info-omits-scattered-meshes` (IN-REVIEW) — omits attached
  `UStaticMeshComponent`s from `scatter_meshes_along_spline`; resolved **docs-only**
  because `actor.get_components` already enumerates those meshes. That escape
  hatch does **not** apply here: no sibling verb exposes a `USplineComponent`'s
  per-point arrive/leave tangents.
- `B-get-splines-info-ignores-spline-mesh` (IN-REVIEW) — class-coverage bug for
  `USplineMeshComponent` roots from `create_spline_mesh_actor` (it adds
  segment-endpoint `startTangent`/`endTangent` for the *spline-mesh* component).
  That is a different component class and a different verb — it does not surface
  per-control-point tangents on an ordinary `USplineComponent`.
- `E-spline-create-actorname-echoes-deduped` (IN-REVIEW) — label-collision
  wrong-target on the create verb. Unrelated.

**Workaround:** infer the tangent took hold from `get_splines_info`'s per-point
`type` (should read `CurveCustomTangent` on the tuned index) plus a `splineLength`
change vs. the pre-tune baseline; accept that the exact tangent vector is not
directly verifiable.

**Fix (proposed):** surface the tangent so it round-trips — either (a) have
`spline.set_spline_point_tangents` **echo** the `arriveTangent`/`leaveTangent` it
applied in its response, and/or (b) add `arriveTangent`/`leaveTangent` to
`get_splines_info`'s per-point `points[]` rows (at least for
`CurveCustomTangent` points) so the applied tangent can be read back
independently. Document whichever lands in `docs/wiki-src/spline.md` (the
`### spline.get_splines_info` / `### spline.set_spline_point_tangents` sections)
so the verify-after-mutate path is discoverable.

severity rationale: impact=readback-omits-a-field (soft — the task's own success
criteria, custom-tangent type + bowed/longer curve, are confirmable today; only
the exact tangent *value* is unverifiable, and inference suffices for the common
question) × reach=rare (hand-tuning spline point tangents is a specialized path,
not an every-session verb) -> Low.

## Evidence (this task — focus `spline.set_spline_point_tangents`)

Cable-tray-route task (5 calls, outcome clean, `friction: none`, no judge ticket
/ `filed_ids` empty; CallAnalyzer trace clean with zero inefficiencies). The
Attempt spawned `CableTrayRoute` (5 control points along X, baseline
`splineLength` 1200, all type `Curve`), applied a strong sideways `+Y 1500`
custom tangent to the middle point (index 2) via `set_spline_point_tangents`, and
read back with `get_splines_info`: the middle point's type flipped to
`CurveCustomTangent`, `splineLength` grew 1200 -> 1764.65, endpoints stayed
`Curve`, points at their placed locations. Every call succeeded first try. The
Prep agent's verbatim friction note:

> "spline.set_spline_point_tangents' response does not echo the applied tangent,
> and no spline readback surfaces tangent vectors (get_splines_info returns only
> index/location/type) — the applied tangent value cannot be verified directly,
> only inferred from the auto-promoted 'CurveCustomTangent' point type and the
> splineLength change."

The task still completed cleanly (the required criteria were confirmable by
inference), so this is a process/discoverability finding surfaced by a clean run,
not a task blocker.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the cable-tray-route task (focus `spline.set_spline_point_tangents`, 5 calls, outcome clean, judge disposition "clean signal — no replay", `filed_ids` empty; CallAnalyzer trace clean, zero inefficiencies). PROCESS finding from Prep's friction note: the applied custom tangent is write-only — `set_spline_point_tangents` does not echo it and `get_splines_info`'s per-point rows carry only `index`/`location`/`type`, so the tangent vector is only inferable (type flip `Curve`->`CurveCustomTangent` + `splineLength` change), never directly verified; a wrong-direction/wrong-magnitude tangent would produce the same type flip and length change. Dedup (ripgrep over OPEN+closed; qmd unavailable): distinct root cause from `E-get-splines-info-omits-scattered-meshes` (attached static meshes; escape via `actor.get_components` — no such sibling verb exists for per-point tangents), `B-get-splines-info-ignores-spline-mesh` (SplineMeshComponent class coverage / segment-endpoint tangents from `create_spline_mesh_actor`, not per-control-point tangents on a USplineComponent), and `E-spline-create-actorname-echoes-deduped` (label collision). No existing ticket covers per-control-point tangent readback. Proposed: echo the applied tangent in `set_spline_point_tangents`' response and/or add `arriveTangent`/`leaveTangent` to `get_splines_info` per-point rows; document in `docs/wiki-src/spline.md`.
