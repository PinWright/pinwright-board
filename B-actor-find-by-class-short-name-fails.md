---
id: B-actor-find-by-class-short-name-fails
title: "`actor.find_by_class` silently returns count:0 for short class names despite documenting them"
status: IN-REVIEW
severity: Medium
category: bug
tags: []
---

# `actor.find_by_class` silently returns count:0 for short class names despite documenting them

`actor.find_by_class` documents (in both the param description and wiki page) that
`className` accepts "Either a short class name (e.g. 'StaticMeshActor') or a full
asset/script path (e.g. '/Script/Engine.StaticMeshActor', '/Game/Foo/BP_Bar')". In
practice, any short class name resolves to nothing and the call returns a clean
`success` with `count:0` and an empty `actors` array — a **silent
success-with-no-effect**. Only the full `/Script/<Module>.<ClassName>` path resolves.

This is the worst failure shape for a discovery method: the caller gets `isError:false`
and a believable, well-formed empty result, so an agent reasonably concludes "there are
no PointLight/StaticMeshActor/DirectionalLight actors in this level" when in fact there
are several. (The only side signal is a `Warning` log line `actor.find_by_class: Class
'<X>' not found`, which the MCP caller never sees.)

Root cause: `Handlers/Actor/QueryHandler.cpp:279-283`. For a non-`/`-prefixed input
the handler calls `FindObject<UClass>(nullptr, *ClassName)`, which does NOT perform a
short-name type lookup (it needs the object already loaded under the right outer); it
returns null for bare class names. The `/`-prefixed branch uses `LoadObject<UClass>`,
which is why full script paths work. This is the same short-name-resolution defect that
`B-asset-list-short-class-ensure`, `B-inspect-class-short-name-fails`, and
`E-class-name-format-inconsistency` already fixed in their handlers by routing through
the shared `ResolveUClass` helper (`Utils/ClassUtils.h`) — but `actor.find_by_class`
was never migrated onto it.

**Repro (live, ExampleProjectWelcome level):**
1. `actor.find_by_class {"className":"StaticMeshActor"}` (the literal wiki example)
   → `{"actors":[],"count":0,"world":"auto","worldPath":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome"}`
2. `actor.find_by_class {"className":"/Script/Engine.StaticMeshActor"}` (same world)
   → `count:2` (UELogo, UELogo2)
3. `actor.find_by_class {"className":"PointLight"}` → `count:0`
4. `actor.find_by_class {"className":"/Script/Engine.PointLight"}` → `count:7`

(Same pattern observed for `DirectionalLight` and `SpotLight` during the originating
task — short name empty, `/Script/Engine.*` path correct.)

**Workaround:** Always pass the full `/Script/<Module>.<ClassName>` path.

**Fix:** Route `className` through `ResolveUClass(ClassName)` (`Utils/ClassUtils.h`)
before the world iteration in `QueryHandler.cpp`, replacing the
`FindObject`/`LoadObject` branch at lines 279-283 — the same migration applied to the
sibling handlers in `E-class-name-format-inconsistency`. Consider also turning an
unresolvable class into a `CLASS_NOT_FOUND` error (or at least surfacing a
`classResolved:false` marker) rather than a silent `count:0`, so callers can
distinguish "class not found" from "zero actors of that class." Check `actor.find_by_tag`
and any other `actor.*` query that resolves a class by name for the same defect.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed during a preview-lighting fuzz task (seed `effect.create_dynamic_light`). After placing PointLight/DirectionalLight/SpotLight actors, `actor.find_by_class` with the documented short names ('StaticMeshActor', 'PointLight', 'DirectionalLight', 'SpotLight') all returned `count:0`; the equivalent `/Script/Engine.*` paths returned the actors. Confirmed root cause at `QueryHandler.cpp:282` (`FindObject<UClass>` instead of `ResolveUClass`). Not a dup: the short-class fixes already on the board (`B-asset-list-short-class-ensure`, `B-inspect-class-short-name-fails`, `E-class-name-format-inconsistency`) covered other handlers and never migrated `actor.find_by_class`.
- `#2-fix` `IN-REVIEW` developer — Migrated `actor.find_by_class` onto the shared `ResolveUClass` helper, matching the sibling handlers fixed in `E-class-name-format-inconsistency`. Replaced the raw `if (ClassName.StartsWith("/")) LoadObject else FindObject` branch (`QueryHandler.cpp:279-283`) with `UClass* ClassToFind = ResolveUClass(ClassName)`, so short names ('StaticMeshActor', 'PointLight') now resolve. An unresolvable class now returns a `CLASS_NOT_FOUND` error (with a guidance message) instead of a silent success-with-`count:0`, so callers can distinguish "class not found" from "zero actors of that class"; the resolve now also runs before world resolution as a cleaner early-exit. Left `actor.find_by_tag` untouched — it matches FName tags, not class names, so it has no class-resolution defect. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Actor/QueryHandler.cpp`. Tests (`Source/EditorAutomationRpcGateway/Private/Tests/World/TestActorHandlers.cpp`): added `FActorFindByClassShortNameResolvesTest` (spawns a real PointLight into the editor world via `actor.spawn`, then queries `actor.find_by_class {"className":"PointLight","world":"editor"}` and asserts success + `count >= 1` — fails if the short name regresses to `count:0`) and `FActorFindByClassUnresolvableErrorsTest` (asserts an unknown class name returns `bSuccess:false` with `ErrorCode == CLASS_NOT_FOUND`). Did not compile/run (later phase).
