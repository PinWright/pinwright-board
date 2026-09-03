---
id: B-set-replication-nonactor-blueprint-noop
title: "misc.set_replication reports requested flags as applied for a non-Actor Blueprint after silently skipping both replication setters"
status: OPEN
severity: Medium
category: bug
tags: [misc, replication, blueprint, wrong-target, false-success]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# `misc.set_replication` does not reject a non-Actor Blueprint

## What happens

The registered contract says the method configures an Actor CDO
(`Handlers/Utility/MiscHandler.cpp:314-320`), but after loading any `UBlueprint` it casts the
generated-class CDO to `AActor` and treats a null cast as a no-op (`:332-351`). It then calls
`Blueprint->Modify()`, marks the Blueprint modified, echoes the requested `replicates` and
`replicateMovement` values, adds only generic asset verification, and sends success at `:353-361`.

Passing a Widget, Object, ActorComponent, or other non-Actor Blueprint therefore dirties the
Blueprint while changing neither flag. The caller observes a normal success payload whose two
booleans came from the request, not from a target CDO.

## Why it matters

This is silent false success, but it requires a wrong Blueprint base class and has a direct
workaround, so the normal High impact is reduced to Medium for reach.

## What should happen

Reject a null `Cast<AActor>` with a typed `INVALID_BLUEPRINT_CLASS` error before `Modify()`. On the
valid path, build the response from `CDO->GetIsReplicated()`/movement-replication readback after the
setters rather than from request locals. Add one Actor-Blueprint success test and one non-Actor
failure test that asserts the Blueprint stays unmodified.

**Workaround:** Use this method only on Actor-derived Blueprints and verify the effective flags with a networking/property reader.

## Related

- Catalog: `wrong-target-scope-or-identity`, `accepted-parameter-silent-noop`, `request-echo-not-result-readback`
- `E-set-property-replicated-no-actor-replicates-flag` — discovery of this method from the networking namespace, not its non-Actor no-op.

## History
- `#1-nonactor-cdo-noop-success` `OPEN` reporter — Traced the method from Blueprint load through
  the null Actor-CDO branch and request-echo response. The separate registration claim that the
  method recompiles was not treated as proof of a runtime failure. No Blueprint was modified.
