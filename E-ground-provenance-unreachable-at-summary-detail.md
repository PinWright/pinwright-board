---
id: E-ground-provenance-unreachable-at-summary-detail
title: "`groundProvenance` is attached only to a per-instance/per-actor row, so it is unreachable at `detail:\"summary\"` and at the default `detail:\"failures\"` when nothing failed — a clean seat is the one case that cannot be proven without re-running the batch at `detail:\"all\"`"
status: OPEN
severity: Medium
category: ergonomic
tags: [spatial, ground_instances, ground_actors, verify_grounding, provenance, detail, readback, batch, response-size]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Provenance matters most when everything looks fine, and that is exactly when it is absent

`groundProvenance` — the block that says *which surface answered the probe* — is reachable only when
a per-subject row is emitted. A batch that seated cleanly emits no rows at the default `detail`, and
none at all at `summary`. So the caller who most needs to know whether they seated onto the landscape
or onto somebody's scatter is the caller with no way to find out short of re-running the whole batch
at `detail: "all"`.

## Confirmed in source

`Handlers/Spatial/GroundPlacementHandler.cpp`:

- Provenance is attached to the **contact object**, at `:524-525`, gated on
  `Report.SupportedColumns > 0` at `:509`.
- The contact object is attached to a **row and only a row**: `:1539` for `spatial.ground_instances`
  (`Row->SetObjectField(TEXT("contact"), GroundRpcContactObject(Result.Seat.Contact))`), and `:580`
  for the actor-side `GroundRpcSeatResultObject`. There is no batch-level contact object anywhere.
- Rows are gated by `detail` at `:1487-1491` —
  `const bool bWantRow = bPlaced ? bIncludeSuccesses : bIncludeFailures;` at `:1487`, `continue` at
  `:1488-1491`.
- `detail` -> flags at `:586-611`: `summary` sets **both** false (`:590-595`), `failures` sets
  successes false (`:596-601`), `all` sets both true (`:602-607`).
- The declared default is `failures` (`:1324-1328`).

Therefore:

| `detail` | outcome | provenance |
|---|---|---|
| `summary` | no rows at all | never |
| `failures` (default) | rows only for non-placed subjects | **absent on a clean batch** |
| `all` | every row | present, and capped at 256 rows |

The gap is not "a level that omits it". It is that **success is the condition under which it is
omitted**. A run where nothing failed and a run where the surface spec was wrong but everything
happened to seat are byte-identical.

## This is not hypothetical — the sibling ticket measured exactly that case

`E-ground-preset-excludes-only-foliage-actors` `#4-ab-on-ten-rocks-count-was-identical` ran the A/B
that proves it: the same ten rocks, once probed against a HISM scatter (seated at Z 562-805, wrong)
and once against the landscape (Z 239-400, correct). Its own headline:

> **BOTH runs reported `placed: 10`.** The correct run and the incorrect run are byte-identical on
> the count field … **The fix's value is entirely in the provenance rows.**

Both of those runs were clean. Under this verb's declared default neither would have emitted a single
row, so the field that is "the only field in either response that distinguishes them" would have been
absent from both.

## Cross-link, stated carefully so this is not read as already served

`E-ground-preset-excludes-only-foliage-actors` `#3-report-names-the-answering-component` built this
block, and its design statement is the *reason* it is per-row:

> **All three verbs get it from one site**, as required: `spatial.verify_grounding`,
> `spatial.ground_actors` and `spatial.ground_instances` all serialize through
> `GroundRpcContactObject`, which emits `groundProvenance` once, gated on `SupportedColumns > 0`.

That one-site design is correct and should not be undone. The single site is
`GroundRpcContactObject`, and a contact object describes **one subject's footprint**, so the block
lands on one subject's row by construction. Nothing in that entry is wrong; it simply never had a
batch-level question to answer. **A fixer who reads only that entry will conclude this is already
served, which is why it is quoted here rather than merely referenced.**

`B-ground-probe-hits-hull-not-render` is *why* provenance exists at all — it is the defect that
makes "which surface answered" a question worth asking, and the reason a green seat is not
self-certifying.

## The three findings compound

Provenance is reachable only at `detail: "all"`, which is:

- the level `E-ground-instances-two-truncation-flags` shows caps `results[]` at 256 rows
  (`GroundPlacementHandler.cpp:64`, `:1492-1496`) and reports that cap under an undocumented flag;
- the level carrying the largest response, on the verb that already spills its uncapped
  `movedInstances[]` receipt (`B-ism-undo-record-unsafe`, sixth-gap encounter).

So proving a clean 400-instance seat requires the most expensive response shape the verb has, and
still loses 144 of the 400 provenance blocks to the row cap. **The one thing a caller cannot do is
ask a cheap question and get an honest answer to it.**

## Ask

**A batch-level provenance summary in the `seat` / `criteria` echo, emitted at every `detail`
level.** The `seat` echo already exists at `GroundPlacementHandler.cpp:1563-1574` and is written
unconditionally at `:1574`, beside `surface` at `:1561` — the natural home.

Shape: for the whole run, how many columns each component class answered. The data is already
aggregated per subject in `FGroundProvenance::SurfaceComponents` (built in
`GroundPlacementUtils.cpp`'s `AggregateColumns`, the reset at `:493-494`); this is a fold of those
per-subject tallies across the batch rather than a new measurement. Something like:

```
seat: { …, provenanceSummary: {
  totalSupportedColumns: 3600,
  byComponentClass: [ {componentClass:"LandscapeHeightfieldCollisionComponent", columns:3591},
                      {componentClass:"HierarchicalInstancedStaticMeshComponent", columns:9} ] } }
```

That makes a clean batch provable at `summary`, and it makes the failure the A/B above found visible
in one number rather than in 400 rows. Keep it a **distribution, not a label**, for the same reason
`#3` gave: a batch that straddles two surfaces must not have an answer invented for whichever half
lost.

Explicitly **not** asked for: moving `groundProvenance` off the row, or emitting rows at `summary`.
Both would undo work that is correct.

## Cross-links

- `E-ground-preset-excludes-only-foliage-actors` (DONE) — `#3` designed the block; `#4` measured the
  case this ticket is about. Quoted above; do not read this as duplicating it.
- `B-ground-probe-hits-hull-not-render` — the underlying defect provenance exists to diagnose.
- `E-ground-instances-two-truncation-flags` (OPEN, Low) — the 256-row cap on the only level that
  emits provenance.
- `B-ism-undo-record-unsafe` (OPEN, High) — response size on the same verb; the sixth-gap encounter.
- `B-ground-instances-default-component-foreign-scatter` (OPEN, Critical) — a batch-level echo
  emitted *before* the mutation would also carry the resolved component, which is that ticket's
  minimum ask. Whoever builds the echo should read both.

## Severity

**Medium.** Impact class, quoted from the rubric: *"a readback omits a field and forces a fallback"*.
That is this exactly — the field exists, it is omitted on the success path, and the fallback is
re-running the batch at `detail: "all"` and paying the largest response the verb produces. It is a
soft blocker: doable, at cost.

**High declined.** High requires the caller to trust a result that is a lie. Nothing here is a lie —
the response omits the field rather than misreporting it, and `detail: "all"` yields the truth. The
A/B in `E-ground-preset-excludes-only-foliage-actors` `#4` shows `placed: 10` is uninformative about
surface choice, but that is that ticket's finding about `placed`, not a false statement made by this
one.

**Low declined.** This is not friction of the "forces a `Read`" kind: no amount of re-reading one
response recovers the field, because it was never emitted. The extra work is a second full batch
against the editor, which is the Medium band's "many extra calls".

**Reach modifier declined.** The three ground verbs are the placement family — used in every
placement session, not in almost every session. No upward bump. Not a rare edge path either: the
affected case is a *successful* seat at the *declared default* `detail`, which is the modal outcome
of the modal call, so no downward bump.

## History
- `#1-provenance-is-row-only` `OPEN` reporter — Re-derived in current source: provenance attached at
  `GroundPlacementHandler.cpp:524-525` gated on `SupportedColumns > 0` (`:509`); contact object
  attached to a row only (`:1539` instances, `:580` actors); rows gated at `:1487-1491`; `detail`
  flags at `:586-611` with `summary` zeroing both (`:590-595`) and `failures` zeroing successes
  (`:596-601`); declared default `failures` (`:1324-1328`). Consequence: a clean batch emits no
  provenance at the default and none at `summary`, so the success path is the unprovable one.
  Corroborated by `E-ground-preset-excludes-only-foliage-actors` `#4`, whose A/B showed a correct and
  an incorrect run identical on `placed` and distinguishable only by the provenance rows — both of
  those runs were clean. Ask is a batch-level fold into the existing `seat` echo (`:1563-1574`), not
  a change to the per-row block, whose one-site design (`#3`) is quoted so it is not undone.
