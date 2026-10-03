---
id: E-actor-not-found-hides-transient-label-match
title: "ACTOR_NOT_FOUND says 'no actor matches by display label' for an RF_Transient editor actor that has exactly that label"
status: OPEN
severity: Low
category: ergonomic
tags: [actor, resolver, error-message, transient]
encounters: 1
lastSeen: 2026-10-02T20:28:11Z
---

# A transient editor actor resolves by object path only, and the refusal does not say so

`McpActorUtils::ResolveActorFiltered` (`Utils/ActorUtils.cpp`) enumerates editor-world label/name
candidates through `UEditorActorSubsystem::GetAllLevelActors()`, which drops `RF_Transient` actors
(and unlisted / non-editable ones) in non-play worlds (`EditorActorSubsystem.cpp:383-388`). That
filter is deliberate (`B-get-components-cannot-resolve-worldsettings` #3 kept label/name
enumeration filtered and added exact object-path resolution for such actors).

The refusal does not reflect it. `ActorNameParamUtils::ResolveActorOrSendError` sends
`ACTOR_NOT_FOUND "No actor matches '<label>' by display label, internal object name, or object
path."` for an actor whose label is exactly `<label>` - the statement is false, and nothing points
the caller at the object path that would work.

Hit by `PinWright.render.capture_actor_preview.EditorActorByNameReportsWorldAndViewport` (run1):
a `SpawnTransientCubeActor` fixture (`RF_Transient`, label set) was refused by label; the test now
passes the object path.

Suggested: when the label/name tiers miss in the editor world, probe the excluded actors for a
label/name match and, on a hit, say that the actor exists but is transient/unlisted and resolves
only by its object path (include the path). No change to what resolves.

## History
- `#1-transient-label-refused` `OPEN` reporter — Filed from the G20 run1 fix round: `render.capture_actor_preview` with `actorPath` = the label of an `RF_Transient` editor-world cube returned `ACTOR_NOT_FOUND` claiming no label match; root cause is `GetAllLevelActors()` dropping transient actors from the label/name tiers, by design, with a message that does not say so.
