---
id: E-actor-get-components-bp-cdo-omits-scs
title: "actor.get_components on a Blueprint asset path returns empty for SCS components — wiki advertises 'a Blueprint CDO' but SCS templates aren't on the bare CDO, steering a set-then-verify into a fruitless read"
status: OPEN
severity: Low
category: ergonomic
tags: [component-readback-routing, actor-get-components, blueprint-scs-get, cdo, scs, readback, docs, discovery]
encounters: 1
lastSeen: 2026-07-01T23:03:42+03:00
---

# `actor.get_components` on a Blueprint asset path returns an empty component list because SCS templates aren't on the bare CDO

`actor.get_components`'s wiki entry (`Docs/wiki-src/actor.md`, `### actor.get_components`)
opens: *"Returns the lean component list for one placed actor **or a Blueprint
CDO**: component name, class path, object path, and scene-component relative
transform fields."* That "or a Blueprint CDO" clause reads as an invitation to
inspect a Blueprint's component layout by handing the verb a `/Game/...` asset
path. In practice, passing a Blueprint asset path resolves the bare generated-class
CDO (`Default__<BP>_C`), and **SCS-defined components are not instantiated on that
CDO** — they are construction-script templates. So the call returns
`{"components":[],"count":0}` for a Blueprint that plainly carries components,
with no signal that the emptiness is a CDO-vs-SCS artifact rather than "this BP
has no components."

This is a **docs-contract gap**: the caller trusts the wiki's "or a Blueprint
CDO" phrasing, gets a confidently-empty result, and has to fall back to
`blueprint.scs.get` to see the real (SCS) component tree. The sibling verb
`actor.describe` **already documents exactly this caveat** in the same file
(`### actor.describe`: *"It does not inspect Blueprint class templates; use
`blueprint.scs.get` or `scs.json` for the class-template component tree."*), but
`actor.get_components` carries no such note and its summary line actively
advertises the Blueprint-CDO path.

## What it should do

Docs/discoverability fix (the data is fully reachable via `blueprint.scs.get`, so
no data loss):

- Add to the `### actor.get_components` overlay in `Docs/wiki-src/actor.md` the
  same caveat `actor.describe` already ships: a **Blueprint asset path returns only
  native / CDO-attached components (CreateDefaultSubobject), NOT SCS-defined
  construction-script components** (the normal way a BP declares components), so
  an SCS-only BP returns an empty list. Route class-template / SCS component
  readback to `blueprint.scs.get` (or `scs.json`). Soften the summary's
  "or a Blueprint CDO" clause accordingly so it does not read as a general
  "inspect a BP's components" affordance.
- Optionally (ergonomic, larger): when handed a BP asset path, merge in the SCS
  component templates so the result matches `blueprint.scs.get`. Docs is the
  minimal, sufficient fix.

## Evidence

Struggle-audit of task `spline.create_spline_mesh_component` (author `BP_FenceRail`,
an Actor BP carrying a `RailMesh` SplineMeshComponent added via SCS; outcome
`done`, 29 MCP calls). During readback verification the agent called
`actor.get_components {actorName:"/Game/FenceSystem/BP_FenceRail"}` and got
`{"components":[],"count":0, ... "Default__BP_FenceRail_C"}` — **empty** — even
though the same BP's `RailMesh` SplineMeshComponent was plainly present via
`blueprint.scs.get`. Agent SAY (transcript line 393): *"actor.get_components on
the BP path returns an empty CDO (SCS components aren't instantiated on the bare
CDO)."* The call log confirms both reads on the same live BP: the empty
`actor.get_components` BP-path read, then `blueprint.scs.get` showing
`ForwardAxis=Y + StaticMesh=SM_Fence_Segment`. So the BP-path
`actor.get_components` is a fruitless corrective step the wiki's "or a Blueprint
CDO" promise invited. (The CallAnalyzer flagged this as a distinct `surprising`
inefficiency alongside the primary diff-only-readback finding filed as
`E-scs-get-diff-only-undocumented`.)

## Distinct from

- `E-scs-get-diff-only-undocumented` (OPEN) — the primary finding from the same
  task: `blueprint.scs.get` elides properties equal to the component CDO default.
  Orthogonal read-side gap on a different verb; that ticket is about a missing
  *property*, this one about a missing *component list* on the BP-CDO path.
- `E-blueprint-get-omits-components-readback-guidance` (IN-REVIEW) — same
  `component-readback-routing` family and the same "route component readback to
  `blueprint.scs.get`" remedy, but a **different method and mechanism**:
  `blueprint.get`'s *registration summary* lies about emitting a `components`
  field the handler never builds. Here the handler genuinely supports a Blueprint
  CDO but the CDO simply has no SCS templates on it, so the gap is the wiki's
  "or a Blueprint CDO" phrasing + a missing SCS caveat on `actor.get_components`.
- `B-get-components-renders-empty-to-caller` (IN-REVIEW) — also renders
  `actor.get_components` empty, but the cause is an MCP transport dead-band spill
  on **large** payloads of a *placed* actor; unrelated to the BP-CDO-vs-SCS
  emptiness here (this response was tiny).

severity rationale: impact=docs/discoverability (data fully reachable via
`blueprint.scs.get`; the empty result is honest-but-misleading given the wiki
promise) × reach=rare (bites only the Blueprint-asset-path sub-mode of
`actor.get_components`; the common placed-actor path is unaffected, and the
sibling `actor.describe` already documents the caveat) -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from a PROCESS struggle-audit of task `spline.create_spline_mesh_component` (author `BP_FenceRail`; outcome done, 29 calls). CallAnalyzer flagged `actor.get_components {actorName:"/Game/FenceSystem/BP_FenceRail"}` returning `{"components":[],"count":0}` (empty CDO) as a `surprising` inefficiency: the `### actor.get_components` wiki entry advertises "the lean component list for one placed actor **or a Blueprint CDO**", but SCS-defined components are not instanced on the bare CDO, so a BP asset path returns empty and the caller must fall back to `blueprint.scs.get`. Agent SAY (line 393): *"actor.get_components on the BP path returns an empty CDO (SCS components aren't instantiated on the bare CDO)."* Docs-only remedy: mirror the caveat `actor.describe` already carries in the same file (`Docs/wiki-src/actor.md`) — a BP path returns only native/CDO-attached components, not SCS templates; route SCS/class-template component readback to `blueprint.scs.get`. Dedup: ripgrep across OPEN/IN-REVIEW/DONE — no ticket covers `actor.get_components`' BP-CDO wiki claim; distinct from `E-scs-get-diff-only-undocumented` (property elision, different verb), `E-blueprint-get-omits-components-readback-guidance` (blueprint.get summary lie, different method/mechanism), and `B-get-components-renders-empty-to-caller` (large-payload transport spill). Seeded `component-readback-routing` family tag for future cross-method aggregation. Severity Low (docs/discoverability; data reachable via scs.get; rare BP-asset-path sub-mode).
