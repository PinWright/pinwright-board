---
id: E-scs-get-no-limit-spills
title: "blueprint.scs.get has no field projection for whole-tree reads — a component-rich Blueprint's full SCS tree overflows the 10k spill threshold and forces a Read of the HttpResponses file"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, scs, response-size, oversized, projection, docs]
encounters: 2
lastSeen: 2026-06-30T15:53:37Z
---

# blueprint.scs.get full-tree reads spill to file with no way to trim per-node bloat

`blueprint.scs.get` already gained the `nameMatch` / `componentClass` *filters*
([E-component-read-filter](E-component-read-filter.md), DONE) for the
**narrow-to-a-few-components** case, and they work cleanly. But those filters do
not help the other dominant use of the verb: the **"read back the whole tree to
confirm the hierarchy"** call — exactly the inspect-then-verify bookends of a
class-level Blueprint edit. When the full component tree is what you actually
want, there is no `limit`, no projection (`fields` / `namesOnly` /
`omitOversized`), and no pagination — so the serialized tree of any
component-rich Blueprint crosses the 10000-char display threshold, comes back as
`outputTooLong`, and the full payload is written to
`Saved/.../HttpResponses/.../<uuid>.json`, forcing the caller to **Read the
spilled file** just to eyeball the hierarchy it was confirming.

This is the same shape as the rest of the `*-no-limit-spills` family
(`E-actor-list-no-limit-spills`, `E-skeleton-list-bones-no-limit-spills`,
`E-volume-get-info-no-limit-spills`, `E-inspect-list-objects-no-limit-spills`,
…): a verbose reader with no narrowing lever for the legitimately-whole read, so
even the natural call overflows. The spill mechanism itself works correctly
(`E-http-response-spill`, DONE); the gap is the missing per-node trim on
`blueprint.scs.get` so a whole-tree confirmation read stays inline.

## What it should do

Mirror the projection levers already shipped / proposed on the sibling readers,
right-sized to the evidence (a confirmation read wants the *hierarchy* — node
name, class, parent — not every template-property delta per node):

- A `fields` / `namesOnly` per-node projection so the common "confirm the tree
  shape" case drops the bulky per-node property payload and returns just
  name / class / parent — the same `fields` allow-list `actor.describe` ships and
  `E-actor-list-no-limit-spills` proposes for `actor.list`. This is the
  highest-value lever here: a hierarchy-confirmation read needs no property bytes.
- Optionally a `limit` + `totalCount` / `truncated` contract (the detectable-
  elision shape `E-recorder-list-sessions-limit` / `E-graph-connections-pagination`
  landed), though for a tree a projection is the better fit than a flat cap.
- **Docs (`docs/wiki-src/blueprint.md`):** note in the `### blueprint.scs.get`
  guidance that an unfiltered whole-tree read on a component-rich Blueprint
  exceeds the inline budget and spills, and that `nameMatch`/`componentClass`
  (already shipped) narrow the *scope* while a `fields`/`namesOnly` projection
  would keep a *whole-tree* confirmation read inline.

## Evidence

From the BP_Light_Bulb_Spotlight class-level build struggle audit (focus
`blueprint.scs.get`, namespace `blueprint`, outcome `clean` — all 12 calls
`ok`/non-error, zero retries). The task's bookend reads were both unfiltered
whole-tree confirmations: step 1 inspected the source
`BP_Light_Bulb_Basic` SCS tree, step 7 read back the full
`BP_Light_Bulb_Spotlight` tree to confirm the brighter point light + new
downward spotlight + hierarchy. Friction note, verbatim: *"the two full-tree
reads exceeded the 10000-char display limit so I read the spilled JSON payloads
from disk."* Two spill+forced-Read events in one task, on a small 3-node-plus
Blueprint — the per-node template-property payload alone pushed each whole-tree
read over budget. The agent narrowed its *verification* reads with the shipped
`componentClass`/`nameMatch` filters (which stayed inline, confirming those work),
but the two reads that legitimately wanted the *entire* tree had no projection
lever and spilled.

## Distinct from

- `E-component-read-filter` (DONE) — added `nameMatch`/`componentClass` *scope*
  filters to `blueprint.scs.get` for the read-a-few-components case. This ticket
  is the complementary gap those filters do not address: the legitimately
  **whole-tree** read still has no per-node trim, so a hierarchy-confirmation read
  overflows. Different lever (projection, not scope), different use case.
- `E-http-response-spill` (DONE) — the generic server-side spill mechanism; this
  ticket is that a specific verbose reader has no narrowing to stay under the
  threshold in the first place (the same relationship every other
  `*-no-limit-spills` ticket has to the spill mechanism).
- `B-scs-get-local-child-parent-link-missing` (IN-REVIEW) — a *correctness* gap in
  how scs.get emits parent/child links; orthogonal to the response-size gap here.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the BP_Light_Bulb_Spotlight class-level build struggle audit (focus `blueprint.scs.get`, outcome clean, 12 calls all `ok`/non-error, zero retries). The two unfiltered whole-tree confirmation reads (step 1 source `BP_Light_Bulb_Basic`, step 7 readback `BP_Light_Bulb_Spotlight`) each exceeded the 10000-char display threshold and spilled to `Saved/.../HttpResponses/.../<uuid>.json`, forcing a Read of the spilled JSON. Friction note verbatim: *"the two full-tree reads exceeded the 10000-char display limit so I read the spilled JSON payloads from disk."* `blueprint.scs.get` already has `nameMatch`/`componentClass` *scope* filters (E-component-read-filter, DONE — and they worked inline on this task's verification reads), but no `fields`/`namesOnly` *projection* or `limit` for the legitimately-whole-tree read, so a hierarchy-confirmation read overflows on any component-rich Blueprint. Proposed: a `fields`/`namesOnly` per-node projection (drop per-node template-property bytes, keep name/class/parent) as the primary lever, optionally `limit`+`totalCount`/`truncated`, and a `### blueprint.scs.get` note in `docs/wiki-src/blueprint.md`. Dedup: ripgrep across OPEN/DONE/WONTFIX — no existing `blueprint.scs.get` spill/limit ticket; `E-component-read-filter` (DONE) owns scs.get *scope* filters not whole-tree projection; `E-http-response-spill` (DONE) is the spill mechanism; the `*-no-limit-spills` family covers the same shape on other readers (actor.list, skeleton.list_bones, volume.get_volumes_info, …). Same fix family, new RPC.
- `#2-single-spline-struct-spill` `OPEN` reporter — Additional evidence, sharper "one heavy struct, not many components" angle: `blueprint.scs.get {blueprintPath:"/App/App/LevelBlueprints/Gates/B_GateArc"}` returned **77,988 chars** and spilled to `Saved/PinWright/HttpResponses/.../<uuid>.json` (repeated across several gates), even though the BP carries just one `GateOpening`-tagged `USplineComponent` plus a `BoxComponent`. The caller's entire data need was "is there a GateOpening-tagged spline + any BoxComponent?" — the lightweight name/class/tags shape this ticket already proposes. Unlike #1 (a component-*rich* whole-tree read), here a **single** component overflows because one `USplineComponent.SplineCurves` struct (position/rotation/scale interp curves with tangents) serializes in full. Source root cause confirmed: `AddOverriddenProperties` (`PinWright_SCSHandlers.cpp:226-271`) exports every CPF_Edit/BlueprintVisible property that differs from the base template via `ExportPropertyToJsonValue` (`:264`), and `SplineCurves` is **not** in the `IsKnownOversizedProperty` omission table — that table (`Utils/PropertyExport.cpp:1176-1184`) holds only `AudioImpulseResponse.ImpulseResponse`, `BodySetup.AggGeom`, and `InstancedStaticMeshComponent.PerInstanceSMData`, so the curve struct emits whole. The only scs.get params are the scope filters `nameMatch`/`componentClass` (`SCSHandler.cpp:48-55`) — no `fields`/`namesOnly` projection. **New fix lever this ticket didn't name:** besides the proposed per-node `fields`/`namesOnly` projection, a smaller targeted fix is to add `/Script/Engine.SplineComponent` → `SplineCurves` to the `IsKnownOversizedProperty` table, which would auto-elide it to a placeholder across `blueprint.scs.get`, `property.list`, and the asset dumpers with no new param. Dedup: this is the same scs.get verbosity/projection gap as #1 (same RPC, same root cause, same primary fix), filed here as additional evidence rather than a separate `E-scs-get-spline-verbose` ticket.
