---
id: B-geometry-target-label-collision-wrong-actor
title: "GeometryTarget resolves only the first matching non-unique actor label, so geometry mutators can edit the wrong mesh and cannot address the intended duplicate"
status: OPEN
severity: High
category: bug
tags: [geometry, actor-resolution, duplicate-label, wrong-target, destructive-mutation, ambiguity, silent-success]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Colliding DynamicMeshActor labels route geometry operations to an arbitrary actor

`GeometryTarget.cpp:23-36` iterates `ADynamicMeshActor` objects, compares only
`GetActorLabel()`, and returns the first match. `FindAnyActor` repeats the same rule at `:39-52`.
The shared `ResolveOrSendError` routes virtually every geometry read and mutator through that helper.
It does not accept an internal object name or object path and does not collect matches to report
ambiguity.

The premise is explicitly guaranteed by UE, not hypothetical: `ActorEditor.cpp:1280-1282` says
actor labels are not supposed to be unique, and `AActor::SetActorLabel` stores the requested label
without uniquing it. `GeometryTarget::Spawn` calls exactly that setter at `:159`, so creating two
geometry actors with the same requested name is sufficient to make the state reachable inside this
module. A later destructive call such as `geometry.delete_triangle`, `boolean_subtract`, or
`convert_to_static_mesh` can succeed against the first actor while the intended actor is untouched;
there is no accepted identifier that can select the second one.

## What should happen

Adopt a deterministic two-pass resolver: exact object path/internal object name first; otherwise
collect display-label matches and refuse with an ambiguity error when more than one exists. Return
the chosen actor's real object path/name in every mutator response. Apply the rule to both
`FindMeshActor` and `FindAnyActor`, including tool/spline operands.

**Workaround:** keep all DynamicMeshActor display labels unique before using `geometry.*`. Unlike
`actor.*`, passing the unique object-name leaf is not a workaround because this helper compares
labels only.

## Related

`E-actor-name-resolution-label-collision` documents how `actor.*` callers can use the unique object
name. It does not cover this separate resolver, where that collision-safe identifier is rejected.

## History
- `#1-source-pattern-scan` `OPEN` reporter — Both geometry resolvers return the first exact display-label match; UE documents labels as non-unique and the module's own spawn helper preserves duplicate labels. Source-only; no actor was created or mutated.
