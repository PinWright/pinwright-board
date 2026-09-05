---
id: E-spawned-audio-component-not-actor-readable
title: "Audio spawn verbs park components on WorldSettings (unresolvable to actor.* reads) — docs/wiki-src/audio.md gives no readback steer to the returned componentPath"
status: IN-REVIEW
severity: Low
category: docs
tags: [audio, spawn, worldsettings, component-path, verification, actor-read, docs]
---

# audio.md gives no readback steer for WorldSettings-owned spawned audio components

`audio.spawn_sound_at_location` and `audio.create_audio_component` create live
`UAudioComponent`s but parent them to the level's `AWorldSettings` info actor
rather than to a discoverable placed actor. `AWorldSettings` is not a placed
level actor, so the `actor.*` read verbs can't resolve it as a read target —
when the caller tries to verify the spawned component's location / sound binding
/ volume through the actor route, every attempt fails:

- `actor.get_component_property` against `WorldSettings_1` (the name reported by
  `actor.list` / `actor.find_by_class`) for `AudioComponent_0/1/2` →
  `[ACTOR_NOT_FOUND] Actor not found: WorldSettings_1` (x3).
- `actor.get_components` on the same WorldSettings actor by **display label**
  (`WorldSettings1`), by **name** (`WorldSettings_1`, alone and with
  `componentClass`), and by **full object path** → all
  `[ACTOR_NOT_FOUND] Actor or Blueprint not found`; supplying only an
  `objectPath` alias with no `actorName` → `[MISSING_REQUIRED_PARAM] Missing
  required parameter 'actorName'`.

So WorldSettings — though it shows up in `actor.list` and resolves under
`actor.find_by_class WorldSettings` (count 1) — is not resolvable as a target for
the actor component-read verbs by *any* of label / name / full path.

**The spawn response already carries the resolvable handle.** Both verbs now
return the spawned component's full object path as `componentPath` (via the shared
`AddComponentVerification`, which emits `componentPath = Component->GetPathName()`
at `Utils/AssetUtils.cpp:951`; `audio.create_audio_component` also sets it directly
at `Handlers/Audio/AudioHandler.cpp:946`). So the readback path is already
unblocked at the API level: feed that returned `componentPath` straight to
`system.inspect.inspect_object` / `property.get` — no reconstruction from
`componentName` + owner and no guessing `AudioComponent_N` indices is needed. The
ticket's original Option #1 ("return a resolvable `componentPath`") was closed by
the sibling [`E-spawn-returns-actor-not-component-path`](E-spawn-returns-actor-not-component-path.md)
fix, which added `componentPath` to the shared helper these audio verbs route
through.

**The residue is a discoverability gap, not an API gap.** The overlay
`docs/wiki-src/audio.md` has no per-method readback guidance, so a caller doesn't
know up front that (a) the owner is the actor-unresolvable WorldSettings and the
`actor.*` read verbs will dead-end, and (b) the correct readback is the returned
`componentPath` fed to `system.inspect.inspect_object` / `property.get`. Without
that steer, the agent burns dead `actor.get_component_property` / `actor.get_components`
calls before falling back to inspecting each component. The original Option #2
("make `AWorldSettings` resolvable to the actor read verbs") is rejected as
over-scoped — it would special-case a non-placed engine singleton in the
load-bearing shared actor-resolution path (`McpActorUtils::FindActorByName`) for a
payoff the already-returned `componentPath` delivers for free.

This is the **process / verification** layer, distinct from the silent-wrong-effect
*tool bug* the judge already filed as
[`B-create-ambient-sound-no-actor`](B-create-ambient-sound-no-actor.md) (that
ticket is `create_ambient_sound` lying about spawning an `AAmbientSound` actor,
now fixed). It is also distinct from
[`E-spawn-returns-actor-not-component-path`](E-spawn-returns-actor-not-component-path.md)
(typed environment spawns — same shared-helper `componentPath` fix, resolvable
owner actor) and [`E-property-route-no-component-path-discovery`](E-property-route-no-component-path-discovery.md)
(guessing a component subobject *name* on a normal actor) — here the owner actor
itself (WorldSettings) won't resolve, and the steer is to bypass the actor route
entirely via the returned `componentPath`.

## What it should do

A caller who spawns an audio component should know, from the method docs, that the
owner is the actor-unresolvable WorldSettings and that the readback path is the
`componentPath` already returned in the spawn response (fed to
`system.inspect.inspect_object` / `property.get`), not the `actor.*` read verbs.

**Fix (docs-only):** add per-method `### audio.spawn_sound_at_location` /
`### audio.create_audio_component` (and `### audio.create_ambient_sound`) H3
sections to `docs/wiki-src/audio.md` documenting that these verbs park the
`UAudioComponent` on the level `AWorldSettings` (which the `actor.*` read verbs
can't resolve), and that the readback/verification path is the returned
`componentPath` → `system.inspect.inspect_object` / `property.get`. No handler or
resolver code change (Option #1 already shipped; Option #2 rejected as
load-bearing over-scope).

**Workaround:** to verify or read a spawned audio component, skip the `actor.*`
read verbs entirely and call `system.inspect.inspect_object` (or `property.get`)
on the `componentPath` returned in the spawn response.

## Evidence (verbatim friction note + call counts)

Friction note: *"the WorldSettings-owned audio components couldn't be read via
actor.get_component_property / actor.get_components by label, name, or full object
path (all ACTOR_NOT_FOUND) -- I had to fall back to system.inspect.inspect_object
on each component's full path to confirm locations/bindings, a
verification-ergonomics gap for spawned-on-WorldSettings audio components."*

Call-log toll on this one task: 3 dead `actor.get_component_property` calls
(`WorldSettings_1` → ACTOR_NOT_FOUND), 5 dead `actor.get_components` variants
(label / name / full path / name+componentClass / objectPath-alias-only →
ACTOR_NOT_FOUND or MISSING_REQUIRED_PARAM), before falling back to 5 successful
`system.inspect.inspect_object` calls (one per component) as the only readback that
worked.

## History
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of an audio-placement task (distinct PROCESS angle from the judge's tool-bug ticket `B-create-ambient-sound-no-actor`, which is the ambient-sound success-with-wrong-effect). The working `audio.spawn_sound_at_location` / `audio.create_audio_component` verbs park their `UAudioComponent`s on `AWorldSettings` and return only a bare `componentName`; the actor read verbs can't resolve WorldSettings by label (`WorldSettings1`), name (`WorldSettings_1`), or full object path (all ACTOR_NOT_FOUND), even though `actor.find_by_class WorldSettings`/`actor.list` see it. Toll: 3 dead `actor.get_component_property` + 5 dead `actor.get_components` variants before falling back to 5 `system.inspect.inspect_object` calls (one per component) as the only readback. Friction note quoted verbatim in body. Proposed: return a resolvable `componentPath` from the audio spawn verbs, and/or make `AWorldSettings` resolvable to `actor.get_components`/`actor.get_component_property`, and/or document the WorldSettings-owner + inspect_object readback path in `docs/wiki-src/audio.md` (tag `docs`). Dedup: ripgrep across OPEN/closed — `B-create-ambient-sound-no-actor` (the ambient bug, judge-filed) is scoped to one verb's wrong effect; `E-spawn-returns-actor-not-component-path` has a resolvable owner actor and is about a missing componentPath on typed env spawns; `E-property-route-no-component-path-discovery` is name-guessing on a normal resolvable actor; `B-get-components-renders-empty-to-caller` (DONE) is the oversized-payload empty-render transport bug. None cover audio components owned by an actor-read-unresolvable WorldSettings.
- `#2-reword-docs-only` `IN-REVIEW` developer — REWORD to docs-only. Verified against current fuzz2 source (HEAD 187db98): the ticket's headline Option #1 ("return a resolvable `componentPath`") is already shipped — `audio.spawn_sound_at_location` (`Handlers/Audio/AudioHandler.cpp:774`), `audio.create_ambient_sound` (`:711`), and `audio.play_sound_attached` (`:508`) all call the shared `AddComponentVerification`, which emits `componentPath = Component->GetPathName()` (`Utils/AssetUtils.cpp:951`, added by the sibling `E-spawn-returns-actor-not-component-path` fix in commit 8be8e5d), and `audio.create_audio_component` sets `componentPath` directly (`Handlers/Audio/AudioHandler.cpp:946`). Option #2 (make `AWorldSettings` resolvable to the actor read verbs) rejected as over-scoped — it would special-case a non-placed engine singleton in the load-bearing shared resolver `McpActorUtils::FindActorByName` for zero marginal benefit over the already-returned `componentPath`. Retitled/reseverity'd (Medium ergonomic → Low docs) and rewrote body + Fix to the only valid residue: the `docs/wiki-src/audio.md` overlay had no per-method readback guidance. Implemented: added `### audio.spawn_sound_at_location` / `### audio.create_audio_component` / `### audio.create_ambient_sound` H3 overlay sections to `docs/wiki-src/audio.md` documenting the WorldSettings owner the `actor.*` verbs can't resolve and steering readback at the returned `componentPath` → `system.inspect.inspect_object` / `property.get`. Regression test: `FWikiHandlerAudioWorldSettingsReadbackTest` (`Private/Tests/Infra/TestWikiHandler.cpp`) renders both method pages via `WikiHandler::RenderPage` and asserts each carries the overlay-exclusive markers (`WorldSettings`, `componentPath`, `system.inspect.inspect_object`) — fails if the H3 sections are reverted (the bare auto summaries name none of them). Mirrors the board's settled docs-only disposition for `E-property-route-no-component-path-discovery` (`FWikiHandlerPropertyComponentDiscoveryTest`).
- `#3-cross-reference-worldsettings-resolver-ticket` `IN-REVIEW` WEAPONS-critic — **Cross-reference only. No claim in this ticket changes, the docs-only fix is not contested, and the status stays `IN-REVIEW`.** A WEAPONS critic review round 4 hit the same `ACTOR_NOT_FOUND` this ticket measured — `actor.get_components` refusing `WorldSettings` by short name and by the full object path `actor.find_by_class` had returned seconds earlier — from a different direction, and the new evidence bears on `#2`'s **Option #2 rejection** rather than on anything this ticket shipped. Two things a reader of this ticket should know. **(1) The rejection's premise does not cover the general case.** `#2` declined to make `AWorldSettings` resolvable on the grounds that "the payoff the already-returned `componentPath` delivers for free" makes the resolver change unnecessary. That holds for components the MCP audio verbs spawn, which is this ticket's whole scope. It does not hold for components the MCP surface never spawned: `UGameplayStatics::SpawnDecalAtLocation` parents every decal to WorldSettings and creates no actor, so a project's game-code decals have **no spawn response and therefore no returned `componentPath`** to steer anyone to, and `actor.get_components` is the only one-call bulk route to them. **(2) The change is smaller than `#2` assumed.** `#2` characterised it as special-casing a non-placed engine singleton inside the load-bearing shared resolver. A source read taken this round says otherwise: the exclusion is not PinWright's. `McpActorUtils::ResolveActorFiltered` builds its editor-world candidate set from `UEditorActorSubsystem::GetAllLevelActors()` (`Utils/ActorUtils.cpp:209`), and it is the **engine** that drops WorldSettings there — `!Actor->IsA(AWorldSettings::StaticClass())`, `C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/Subsystems/EditorActorSubsystem.cpp:388`. The resolver's own explicit-world and PIE branches use `TActorIterator` (`Utils/ActorUtils.cpp:176`), which has no such exclusion, as does `actor.find_by_class` (`Handlers/Actor/QueryHandler.cpp:467`) — which is why that verb returns the actor this one cannot resolve. So aligning the editor branch onto `TActorIterator` is a candidate-set consistency fix across three branches of one function, not a WorldSettings special case. Filed as `B-get-components-cannot-resolve-worldsettings` (OPEN, High) rather than reopening this ticket, because this one is a docs ticket about audio verbs, its shipped fix is correct within its scope and should not be blocked, and the new evidence contests a disposition rather than a fix. No change is requested here; a reader arriving at `#2`'s Option #2 rejection should read that ticket before treating the question as settled.
