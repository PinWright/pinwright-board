---
id: E-spawn-no-scale-param
title: "actor.spawn / actor.spawn_from_blueprint take location + rotation but no scale, forcing a second set_transform for any non-unit prop"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [actor, spawn, transform, scale, ergonomics, docs]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# actor.spawn has no scale parameter — every non-unit prop spawn costs a follow-up set_transform

`actor.spawn` declares exactly three placement params —
`location` ({x,y,z}), `rotation` ({pitch,yaw,roll}), plus the class/mesh/name
inputs — and **no `scale`**. The handler reads only Location and Rotation
(`SpawnHandler.cpp` lines 58-59 declare the params; lines 67-68 read
`GetVector("location")` / `GetRotator("rotation")`; the spawn then calls
`SetActorLocationAndRotation(Location, Rotation, ...)` with no scale path).
`actor.spawn_from_blueprint` has the identical surface (lines 241-256). So the
spawned actor always lands at unit scale `(1,1,1)`, and any intent that wants a
scaled prop — a wide-short plinth, a stretched wall, a tall column, an enlarged
loot orb — must immediately follow the spawn with a separate
`actor.set_transform` carrying the scale.

A spawn-then-scale is a single staging intent ("place a wide short plinth"), but
the tool surface splits it into two calls. `location` and `rotation` are already
first-class spawn params; `scale` is the missing third component of the same
transform, and it is just as commonly non-default at spawn time as the other two.

**Evidence — this task (loot-pedestal staging, outcome `clean`):**
The agent spawned the cube pedestal (call #12, `actor.spawn Cube at (500,0,0)`),
then had to issue a dedicated second call (#14, `actor.set_transform cube scale
(2,2,0.5)`) purely to make it the "wide, short plinth" the task asked for — a
two-call sequence for one spawn intent. The friction note names this as the only
non-obvious part of the whole task (verbatim):

> "the only non-obvious bit (actor.spawn has no scale param) was resolved cleanly
> by setting scale via actor.set_transform after spawn, exactly as the wiki
> implies."

It resolved cleanly, but only because the agent already knew to reach for
`set_transform` — the cost is the extra round-trip on an every-session verb.

**Discoverability angle (docs):** the wiki has **no `### actor.spawn` section at
all** — `docs/wiki-src/actor.md` documents `actor.spawn_from_blueprint`,
`actor.add_component`, etc. in dedicated sections but only *mentions* `actor.spawn`
in the overview bullet (line 11) and cross-refs. Nothing tells a caller that
`location`/`rotation` are settable at spawn but `scale` is not, so "spawn already
scaled?" is answered only by inspecting the param schema or trial-and-error. Even
if the scale param is not added, the page should state the gap and the
spawn→`set_transform` recipe explicitly.

**Fix (ergonomic, additive):** add an optional `scale` ({x,y,z}, defaults to
`(1,1,1)`) param to `actor.spawn` and `actor.spawn_from_blueprint`, applied in
the same place the spawn already sets location/rotation (e.g. build an
`FTransform(Rotation, Location, Scale)` and use the location-rotation-scale
setter). This collapses the common spawn-scaled-prop intent back to one call and
makes `scale` symmetric with the `location`/`rotation` params already present.

**Docs (`docs/wiki-src/actor.md`, tagged `docs`):** add a dedicated
`### actor.spawn` section that lists its params and — until/unless `scale` is
added — states that scale is not settable at spawn and points at the
`actor.set_transform` follow-up.

**Workaround:** spawn, then call `actor.set_transform` on the returned actor name
with the desired `scale`. Clean and reliable, but one extra call per scaled
spawn.

## History
- `#2-spawn-scale-param` `IN-REVIEW` developer — Added an optional `scale` ({x,y,z}, default `(1,1,1)`) param to both `actor.spawn` and `actor.spawn_from_blueprint`, read via `Ctx.GetVector("scale", FVector::OneVector)` and applied post-spawn with `SetActorScale3D(Scale)` (mirrors the proven `actor.set_transform` scale path), so a non-unit prop spawns in one call instead of a follow-up `set_transform`. Files: `Source/PinWright/Private/Handlers/Actor/SpawnHandler.cpp` (both verbs — param decl, read, apply). Docs: added a `### actor.spawn` section to `Docs/wiki-src/actor.md` listing location/rotation/scale and the one-call scaled-spawn recipe (coordinate with `E-effect-actor-name-slot-vs-actorname`, which also plans an `### actor.spawn` section — augment, don't duplicate). Regression test: `PinWright.actor.spawn.AppliesScale` in `Source/PinWright/Private/Tests/World/TestActorHandlers.cpp` spawns a real PointLight with `scale {2,3,0.5}` through the production handler and asserts the spawned actor's world `GetActorScale3D()` matches (and that both verbs register a `scale` param); it would fail if the apply path were reverted (actor would land at unit `(1,1,1)`).
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of a loot-pedestal staging task (outcome `clean`, friction note "none — the wiki was clear"). Even on the clean path, `actor.spawn` declares only `location`+`rotation` (no `scale`), so the wide-short-plinth intent took two calls: spawn cube at (500,0,0) (call #12) then a dedicated `actor.set_transform cube scale (2,2,0.5)` (call #14). The agent's friction note flags the missing scale param as the task's only non-obvious bit. Confirmed in source: `Source/PinWright/Private/Handlers/Actor/SpawnHandler.cpp` declares `location`/`rotation` params (lines 58-59) and reads only those (lines 67-68); `actor.spawn_from_blueprint` is identical (lines 241-256). Wiki has no `### actor.spawn` section to document the gap. Proposed: add optional `scale` param to both spawn verbs (symmetric with location/rotation) and a docs section. Dedup: ripgrep across OPEN/closed found no spawn-scale ticket — `E-spawn-returns-actor-not-component-path` is about typed `environment.spawn_*` verbs withholding componentPath (different verbs, different gap); the `E-spawn-category-*` tickets are about a spawn_category concept, not scale; no set_transform ticket covers this.
