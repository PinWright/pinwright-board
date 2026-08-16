---
id: B-derived-state-report-contract-gaps
title: "Small contract gaps in the new derived-state verbs: empty authoritativeVerb, width verb named for a depth write, remedy/notRefreshed contradiction, missing snake_case alias"
status: OPEN
severity: Low
category: bug
tags: [water, spline, material-authoring, derived-state, wire-contract, conventions]
---

# Five small gaps in the derived-state changeset's wire contract

None of these produce a false success — the changeset's core honesty property holds. They are
places where the structured payload is less useful than the prose beside it, or where a convention
is missed. Grouped into one ticket because they are all one edit each in the same two files.

## 1. A refusal can name no verb at all

`Handlers/Geometry/SplineHandler.cpp` — `OutAuthoritativeVerb` defaults to empty and is set only for
`UWaterSplineComponent`. `docs/rpc-design.md` §5a's own rule is *"A refusal must name a verb that
exists"*, and §5a's **other** rule ("ask the engine, do not maintain a type list") is exactly what
makes the empty branch reachable: any future spline type that overrides
`AllowsSplinePointScaleEditing()` to false gets `authoritativeVerb: ""`. The generic `explanation`
prose is fine, but the machine-readable field is the one that is supposed to survive.

Suggested: when no verb is known, say so in the field rather than leaving it empty, and name the
generic route (`property.set` on the authoritative component, or the discovery page).

## 2. `authoritativeVerb` names the width verb even for a `Scale.Y` (depth) write

Same file. `Docs/wiki-src/spline.md` tells callers "`authoritativeVerb` is the verb to call
instead", so a depth write is pointed at `water.set_river_width_at_spline_point`. The `explanation`
string does mention both verbs, so a human recovers — but §5a's whole argument for the structured
block is that recovery should not require parsing prose.

Suggested: pick the verb from which scale component the caller actually changed.

## 3. `remedy` / `notRefreshed[]` contradict the header's documented contract

`Utils/DerivedStateReport.h` documents `Remedy` as *"What a caller should do about anything in
`NotRefreshed`. Empty when the coverage is complete."* But
`Handlers/Material/MaterialAuthoringHandler.cpp` **always** appends one `NotRefreshed` entry ("open
asset editors, which keep their own preview material state") while setting `Remedy` only when
`IsComplete()` is false — and then the remedy names `landscape.set_material`, which does nothing
about asset editors. Net wire shape on a clean run: `complete: true` + non-empty `notRefreshed[]` +
no `remedy`, which reads as a contradiction to anyone holding the header's contract.

The entry is also appended when `bMeasured == false`, i.e. a run that never enumerated consumers
still enumerates a gap.

Suggested: either separate "out of scope by design" from "in scope and missed" as two fields, or
relax the header's documented contract to match the behaviour. The behaviour is defensible; the
documentation of it is not.

## 4. `pointIndex` has no `point_index` alias

`Handlers/Water/WaterHandler.cpp` — `Ctx.GetIntFirstOf({ TEXT("pointIndex") })`. `CLAUDE.md`
Conventions: *"handlers accept both `camelCase` and `snake_case` variants"*. `GetIntFirstOf` takes a
list precisely for this. A `point_index` caller is silently dropped, then trips the
`pointIndex` XOR `allPoints` check and gets `INVALID_PARAMS` — safe, but an avoidable rejection with
a misleading message. Verified directly.

## 5. Fixed `0.01` apply tolerance against `float` storage

`Handlers/Water/WaterHandler.cpp` stores `static_cast<float>(Value)` and compares the read-back
against a fixed `0.01` tolerance. Above roughly 65,000 the float ULP exceeds 0.01, so a **correct**
write would report `APPLY_FAILED`. Realistic river widths sit far below that, so this is a latent
cliff rather than a live bug; a relative tolerance removes it.

## Also: citation drift in the new comments and docs

Harmless, but this codebase argues from `file:line` evidence, so drift costs the next reader time.
Reported: `WaterSplineComponent.h:54` should be `:53` (in `rpc-design.md` §5a and
`SplineHandler.cpp`); `WaterSplineComponent.h:62` should be `:60` (`WaterHandler.cpp`);
`LandscapeEdit.cpp:710` is cited for `GetCombinationMaterial`, which is at `:585-630`.
`DerivedStateReport.h`'s own "Emits `consumerRefresh: {...}`" comment omits
`consumersWithNothingToRefresh`, which the function does emit.

## History
- `#1-found-by-audit` `OPEN` reporter — Found by auditing the derived-state-honesty changeset
  (`099b83b3`..`0fe35187`) against `Docs/rpc-design.md` during integration pass 11. Items 4 and the
  `SafePoint` question were re-verified directly by the integration pass; items 1-3, 5 and the
  citation drift are as reported by the audit and are **not** independently re-verified.
