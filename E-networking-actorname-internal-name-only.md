---
id: E-networking-actorname-internal-name-only
title: "networking.* actor verbs (set_owner / get_networking_info / check_has_authority) resolve actorName by internal object name ONLY and reject the display label — yet set_owner's own success response echoes actorName=<label>, so re-using the value the call reports back gets [NOT_FOUND]"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [networking, set_owner, get_networking_info, check_has_authority, actorname, display-label, internal-name, not-found, cross-method-consistency, self-contradicting-echo]
---

# The networking namespace's actor-targeting verbs accept only the internal object name in `actorName`, refuse the display label, and `set_owner` reports the rejected label back as `actorName` on success

The actor-targeting verbs in the `networking` namespace —
`networking.set_owner`, `networking.get_networking_info` (actor branch),
`networking.check_has_authority`, `networking.check_is_locally_controlled` —
resolve their `actorName` slot through a **private local helper** that matches
the **internal object name only**, by exact `GetName()` equality, with no label
and no path fallback:

```cpp
// NetworkingHandler.cpp:62-69
static AActor* FindActorByName(UWorld* World, const FString& ActorName)
{
    if (!World) return nullptr;
    for (TActorIterator<AActor> It(World); It; ++It)
    {
        if (It->GetName() == ActorName) return *It;   // GetName() ONLY — no label, no path
    }
    return nullptr;
}
```

This is the inverse of `editor.focus_actor` (label-only, see
`E-focus-actor-rejects-internal-name-label-only`) and is inconsistent with the
per-actor `actor.*` verbs, whose shared `McpActorUtils::FindActorByName`
(`Utils/ActorUtils.cpp`) matches label **OR** `GetName()` **OR** path (the
basis of `E-actor-name-resolution-label-collision`). So three actor-resolution
rules now coexist in the same product: `actor.*` = label-or-name-or-path,
`editor.focus_actor` = label-only, `networking.*` = internal-name-only — and a
caller cannot tell which from the docs (the `set_owner` wiki param help is just
"Name of the actor", `NetworkingHandler.cpp:730`).

## The quotable self-contradiction

The acute part is that `set_owner`'s **own success response reports the display
label in a field literally named `actorName`** — the exact string it rejects as
*input* `actorName`. `AddActorVerification` (`NetworkingHandler.cpp:754`) fills
the verification block from the resolved actor, so it emits
`"actorName":"TurretBase"` (the label) even though the call only succeeded
because the *internal* name `StaticMeshActor_4` was passed. An agent that reads
the success payload and re-uses its `actorName` value verbatim — the obvious
round-trip — gets `[NOT_FOUND] Actor not found` for an actor that demonstrably
exists and was just operated on. The error text ("Actor not found") also hides
the real cause (wrong identifier *kind*, not a missing actor), the same
confusing-error shape flagged for `focus_actor`.

This is distinct from the existing tickets:
- `E-actor-name-resolution-label-collision` (IN-REVIEW) — the **`actor.*`**
  verbs, which *do* accept the internal name; that ticket documents it as the
  collision-safe key. The networking verbs **only** accept it and refuse the
  label outright — opposite acceptance gap, different handler/helper.
- `E-focus-actor-rejects-internal-name-label-only` (OPEN) — the *inverse*
  resolution rule on a different method (`editor.focus_actor`, label-only). This
  is its networking-namespace twin in the opposite direction (internal-name-only).
- `E-networking-wiki-stub-no-readback-contract` (OPEN) — about the networking
  wiki being a stub and `get_networking_info`'s *return shape* being
  undocumented. This ticket is about the **input-side** `actorName` *value*
  resolution (label rejected) and the self-contradicting *output* echo, not the
  readback field set.
- The `name`/`actorName`/`systemName`/`label` param-**key** drift family
  (`E-effect-actor-name-slot-vs-actorname`, `E-geometry-create-name-vs-actorname`)
  — those are about which *key* the param uses; here the key `actorName` is fine,
  it is the accepted *value* (label vs internal name) that diverges.

## What it should do

Pick ONE and document it:

1. **Preferred — match the `actor.*` verbs:** replace the local
   `NetworkingHandler.cpp:62-69` `FindActorByName` with the shared
   `McpActorUtils::FindActorByName` (label OR internal name OR path), so the
   identifier a caller already holds from `actor.spawn`/`actor.list` (or that
   `set_owner` itself echoes back) just works. This removes the
   inverse-of-`actor.*` surprise and the self-contradicting echo in one change.
2. **If internal-name-only is intentional:** make the error self-explaining
   (e.g. `[NOT_FOUND] No actor with internal object name '<X>' (this method
   matches the internal object name from actor.list's 'name' field, not the
   display label)`), stop echoing the display label as `actorName` in the
   success verification (report the resolved internal `GetName()` instead, or
   add a separate `label` field), and document the internal-name-only rule on
   `docs/wiki-src/networking.md`.

## Evidence (replay-confirmed, live, ExampleProjectWelcome)

Spawned two StaticMeshActors with labels `TurretBase` / `OwningPlayerProxy`
(internal names `StaticMeshActor_4` / `StaticMeshActor_5`, confirmed via
`actor.list filter=TurretBase namesOnly` → `{"label":"TurretBase","name":"StaticMeshActor_4",...}`).

1. `networking.set_owner {actorName:"TurretBase", ownerActorName:"OwningPlayerProxy"}`
   (the display labels `actor.spawn` accepted as `actorName`)
   → **`[NOT_FOUND] Actor not found`**
2. `networking.set_owner {actorName:"StaticMeshActor_4", ownerActorName:"StaticMeshActor_5"}`
   (the internal names) → **success**, response:
   ```json
   {"success":true,"message":"Set owner of StaticMeshActor_4 to StaticMeshActor_5",
    "actorPath":".../PersistentLevel.StaticMeshActor_4","mapPath":"/Game/Maps/ExampleProjectWelcome",
    "actorName":"TurretBase","actorGuid":"ADE83E62...","existsAfter":true,"actorClass":"StaticMeshActor"}
   ```
   Note `"actorName":"TurretBase"` — the very label rejected as input in step 1.
3. `networking.get_networking_info {actorName:"TurretBase"}` → **`[NOT_FOUND] Actor not found`**
4. `networking.check_has_authority {actorName:"TurretBase"}` → **`[NOT_FOUND] Actor not found`**

To rule out a colliding-label confound, a **uniquely-labeled** actor
`UniqueTurretXYZ` (internal `StaticMeshActor_6`) was spawned: both
`networking.set_owner {actorName:"UniqueTurretXYZ", ...}` and
`networking.get_networking_info {actorName:"UniqueTurretXYZ"}` still returned
`[NOT_FOUND]` — so it is not label ambiguity; the networking verbs simply never
resolve display labels at all.

Originating task friction note (verbatim): *"Minor: networking.set_owner
rejected the display labels (\"TurretBase\"/\"OwningPlayerProxy\" that
actor.spawn accepted as actorName) with \"[NOT_FOUND] Actor not found\" — it
only resolves the internal actor name (StaticMeshActor_0/_2 from actor.list's
name field), unlike actor.spawn/actor.list which accept/return labels; the
set_owner wiki just says \"Name of the actor\" without flagging
label-vs-internal-name, so one call had to be retried with the internal
names."*

**Workaround:** Pass the **internal object name** (the `name` field from
`actor.list` / the object-path leaf, e.g. `StaticMeshActor_4`) to every
`networking.*` actor verb — never the display label, and never the `actorName`
value that `set_owner`'s own success response echoes back.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed during a multiplayer turret-ownership task (seed `networking.set_owner`). The networking namespace's actor-targeting verbs (`set_owner`, `get_networking_info` actor branch, `check_has_authority`, `check_is_locally_controlled`) resolve `actorName` via a private `NetworkingHandler.cpp:62-69` `FindActorByName` that matches `GetName()` ONLY — display label and path are rejected with `[NOT_FOUND] Actor not found`. Replay: `set_owner {actorName:"TurretBase"}` (the label `actor.spawn` accepted) failed `[NOT_FOUND]`; `{actorName:"StaticMeshActor_4"}` (internal name) succeeded but its verification block echoed `"actorName":"TurretBase"` (the rejected label); `get_networking_info`/`check_has_authority` on the label also failed `[NOT_FOUND]`; a uniquely-labeled actor still failed, ruling out label collision. This is the internal-name-only inverse of `E-focus-actor-rejects-internal-name-label-only` (label-only on `editor.focus_actor`) and inconsistent with the `actor.*` verbs' shared label-OR-name-OR-path resolver (`E-actor-name-resolution-label-collision`). Distinct from `E-networking-wiki-stub-no-readback-contract` (return-shape docs gap, not input value resolution) and the `name`-vs-`actorName` param-key drift family (key, not value). Dedup: ripgrep over OPEN+closed (qmd unavailable) found no existing ticket on the networking-namespace label-rejection or the self-contradicting `actorName` echo. Proposed: swap the local helper for `McpActorUtils::FindActorByName` (label OR name OR path) to match `actor.*`, OR make the error self-explaining, stop echoing the label as `actorName` in the success verification, and document the internal-name-only rule on `docs/wiki-src/networking.md`.
- `#2-retriage` `OPEN` triage — Low→Medium: set_owner echoes a label as actorName that its own input rejects, so the obvious round-trip gets [NOT_FOUND], niche networking path.
- `#3-route-shared-resolver` `IN-REVIEW` developer — Option 1 (Preferred), matching the `editor.focus_actor` precedent (`E-focus-actor-rejects-internal-name-label-only`) and keeping the `actorName=GetActorLabel()` echo convention untouched. Replaced the private `NetworkingHelpersNew::FindActorByName` body (`NetworkingHandler.cpp:62-70`, `GetName()`-only, case-sensitive) with a one-line forward to the shared `McpActorUtils::FindActorByNameSimple` (label OR internal name OR object path, case-insensitive; honors the passed `World` — every networking call site passes `GEditor->GetEditorWorldContext().World()`). All four actor verbs (`set_owner`, `check_has_authority`, `check_is_locally_controlled`, `get_networking_info` actor branch) call through the same local helper, so they all now accept the display label that `actor.spawn`/`actor.list` accept and that `set_owner`'s own success verification echoes back — the self-contradicting round-trip is gone. Files: `Source/PinWright/Private/Handlers/Networking/NetworkingHandler.cpp` (added `#include "Utils/ActorUtils.h"`; rewrote the helper). Regression test: `PinWright.networking.set_owner.ResolvesDisplayLabel` in `Source/PinWright/Private/Tests/Networking/TestNetworkingHandlers.cpp` — spawns two actors with display labels deliberately distinct from their internal `GetName()`, drives the production `networking.set_owner` handler by the LABELS, and asserts success + `Probe->GetOwner()==OwnerProbe` + the echoed `actorName` equals the resolved label. Reverting the helper to `GetName()`-only fails the success assertion (the distinct label never equals `GetName()`, so the handler returns `[NOT_FOUND]`).
