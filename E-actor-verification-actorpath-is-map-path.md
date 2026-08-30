---
id: E-actor-verification-actorpath-is-map-path
title: "AddActorVerification's `actorPath` result field reports the map PACKAGE path, not the actor's object path — useless for disambiguating colliding labels, on ~120 actor-mutating verbs"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [actor, add_tag, remove_tag, set_transform, verification, actorpath, misreport, shared-helper, AddActorVerification]
---

# `actorPath` in the standard actor-verification block is the map package path, not the actor path

Every actor-mutating verb that emits the standard verification block (`add_tag`,
`remove_tag`, `set_transform`, `set_visibility`, `attach`, `detach`, the
`property` operators, `spawn`, `duplicate`, all `volume.*`/`lighting`/`geometry`
spawners — ~120 call sites) returns a result field literally named **`actorPath`**
whose value is the actor's **map/world PACKAGE path** (`/Game/Maps/<MapName>`),
**not** the actor's object path
(`/Game/Maps/<MapName>.<MapName>:PersistentLevel.<ObjectName>`).

The sibling identity fields in the same block are correct (`actorName` = label,
`actorGuid` = the actor GUID), so the call works and the actor is right — but a
field named `actorPath` that hands back a path shared by *every* actor in the map
is a concrete misreport. It is most harmful in exactly the situation where you'd
reach for it: when display labels collide. In the audited task, all seven
PointLights share the label `TestPointLight`; the one field that should tell the
caller *which* of the seven just got tagged (`actorPath`) instead returns the map
path common to all of them. The caller cannot use it to confirm the mutation
landed on the intended actor (must fall back to `actorGuid` or a follow-up
`find_by_tag`/`get`).

This also contradicts the engine-wide spelling: `actor.spawn`/`actor.duplicate`
are documented (see `E-actor-verbs-reject-actorpath-slot`) as returning the
spawned actor's identity under `actorPath` via `Spawned->GetPathName()` — i.e. the
real object path. So `actorPath` means *object path* in the spawn-response prose
yet means *map package path* in this verification helper, on the same surface.

## Root cause (source)

`Source/PinWright/Private/Utils/AssetUtils.cpp`,
`AddActorVerification()`:

```cpp
// AssetUtils.cpp:835-836
FString ActorPath = Actor->GetPackage() ? Actor->GetPackage()->GetPathName() : Actor->GetPathName();
Response->SetStringField(TEXT("actorPath"), ActorPath);
```

`Actor->GetPackage()->GetPathName()` is the package (map) path; the real actor
object path (`Actor->GetPathName()`) is only used as the no-package fallback,
which a placed actor never hits. The correct value is right there in the sibling
helper in the same file, `AddChainableActorFields()` (AssetUtils.cpp:847):

```cpp
Response->SetStringField(TEXT("actorPath"), Actor->GetPathName());
```

So one file fills the same-named field two different ways; the verification path
picked the package path.

## Repro (verbatim, live, ExampleProjectWelcome — seven PointLights all labelled "TestPointLight")

`actor.add_tag {"actorName":"TestPointLight","tag":"AmbiguityProbe"}` →
```json
{"wasPresent":false,"actorName":"TestPointLight","tag":"AmbiguityProbe",
 "actorPath":"/Game/Maps/ExampleProjectWelcome","actorGuid":"D25C83DF41A7DE487325D79364C4CFE8",
 "existsAfter":true,"actorClass":"PointLight"}
```
`actor.remove_tag {"actorName":"TestPointLight","tag":"AmbiguityProbe"}` →
```json
{"wasPresent":true,"actorName":"TestPointLight","tag":"AmbiguityProbe",
 "actorPath":"/Game/Maps/ExampleProjectWelcome","actorGuid":"D25C83DF41A7DE487325D79364C4CFE8",
 "existsAfter":true,"actorClass":"PointLight"}
```

For comparison, `actor.find_by_class {"className":"/Script/Engine.PointLight"}`
and `actor.find_by_tag` report the **real** actor object path for the same actor:
`/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.PointLight_1`.
A confirming `find_by_tag {"tag":"AmbiguityProbe"}` showed exactly one actor
(`PointLight_1`) carried the probe tag — so `add_tag` resolved the ambiguous
label to `PointLight_1` correctly, but its `actorPath` field
(`/Game/Maps/ExampleProjectWelcome`) names the map, not `PointLight_1`.

## What it should do

Set the `actorPath` field in `AddActorVerification` to `Actor->GetPathName()`
(the actor object path), matching the sibling `AddChainableActorFields`, the
spawn/duplicate response spelling, and the path that `find_by_class`/`find_by_tag`
already return — so callers can identify *which* actor was mutated when labels
collide. (If the map package path is also wanted, surface it under a distinctly
named field such as `mapPath`/`levelPath`, not `actorPath`.) This is a
one-line-per-helper change in a shared utility and propagates to all ~120 call
sites that emit the verification block.

**Workaround:** Use `actorGuid` (correct) from the verification block, or re-query
with `actor.find_by_tag` / `actor.get` to obtain the real actor object path.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed during a lighting-cleanup
  tag-grouping task (seed `actor.find_by_tag`; culprit `actor.add_tag` /
  `actor.remove_tag`). The attempt's friction note flagged that all seven
  PointLights share the label "TestPointLight" so label-based `actorName` is
  ambiguous; investigating that, found the deeper misreport: the verification
  block's `actorPath` returns the map package path
  (`/Game/Maps/ExampleProjectWelcome`), not the actor object path, so it cannot
  disambiguate which colliding-label actor was touched. Root cause confirmed in
  source: `AddActorVerification` (AssetUtils.cpp:835-836) sets `actorPath` =
  `Actor->GetPackage()->GetPathName()`, while the sibling `AddChainableActorFields`
  (AssetUtils.cpp:847) correctly uses `Actor->GetPathName()`; the helper is shared
  across ~120 actor-mutating call sites (add_tag/remove_tag/set_transform/
  set_visibility/attach/detach/property/spawn/duplicate/volume.*/lighting/geometry).
  Dedup (ripgrep over OPEN+closed; qmd unavailable): distinct from
  `E-actor-verbs-reject-actorpath-slot` (OPEN — that is the INPUT-key drift:
  `actor.*` consumers rejecting `actorPath` as an input alias; this is the OUTPUT
  field's value being wrong) and from `E-inspect-find-by-tag-internal-name-not-label`
  (OPEN — `name` field internal-name-vs-label on the inspect twins, not `actorPath`).
  No existing ticket covers the verification block's `actorPath` value. Proposed:
  set `actorPath` to `Actor->GetPathName()` in `AddActorVerification` to match the
  sibling helper and the spawn-response spelling.
- `#2-fix` `IN-REVIEW` developer — Fixed `AddActorVerification`
  (`Source/EditorAutomationRpcGateway/Private/Utils/AssetUtils.cpp`, now at
  :926-940): `actorPath` now uses `Actor->GetPathName()` (the actor OBJECT path),
  matching the sibling `AddChainableActorFields`, the spawn/duplicate responses,
  and the `find_by_*` verbs — so it can disambiguate which colliding-label actor
  was mutated. The map/world PACKAGE path it previously returned is preserved but
  re-surfaced under a distinct `mapPath` field (only when the actor has a package).
  This one-helper change propagates to all ~127 actor-mutating call sites that emit
  the verification block, and as a side effect makes spawn/duplicate's own
  pre-`AddActorVerification` `actorPath = GetPathName()` write idempotent instead of
  clobbered (confirmed at `SpawnHandler.cpp:214` then `:226`). Note line citations
  in the body/History #1 (AssetUtils.cpp:835-836/:847) were stale; actual lines are
  :926-940/:938 — code was byte-identical to what was quoted.
  Regression test: `TestActorHandlers.cpp` →
  `EditorAutomationRpcGateway.actor.add_tag.VerificationActorPathIsObjectPath`. It
  spawns a real PointLight via the production `actor.spawn`, reads the live actor's
  true object path and package path from the world, drives the ticket's repro verb
  `actor.add_tag`, and asserts the verification block's `actorPath` equals the
  object path and is NOT the package path, plus that the new `mapPath` equals the
  package path. Reverting the fix (setting `actorPath` back to the package path)
  fails the test. Not compiled/run here (a later phase verifies green).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
