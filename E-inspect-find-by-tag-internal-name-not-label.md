---
id: E-inspect-find-by-tag-internal-name-not-label
title: "system.inspect.find_by_tag / find_by_class / list_objects omit the display label, so a read-only audit can't read it without re-calling the actor.* write twin"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [inspect, find_by_tag, actor, display-label, read-only-audit, name-field]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# The read-only `system.inspect` enumerators surface only the internal object name, not the display label

The read-only inspector `system.inspect.find_by_tag` is documented as the
**"Read-only counterpart to actor.find_by_tag"** (its wiki page), and
`actor.find_by_tag.md` actively steers audit-intent callers to it: *"For
read-only audit prefer system.inspect.find_by_tag."* So an agent doing a
read-only tag audit (the exact task here) follows the docs to the `inspect`
variant — and then can't read the user-facing display label at all, because the
inspect rows carry only the internal object name.

The two siblings disagree on what the `name` field *means*, on an identical
query against identical world state:

- `actor.find_by_tag` (the write-namespace twin) returns the **user-facing
  display label** in `name`.
- `system.inspect.find_by_tag` (the read-only twin the docs recommend) returns
  the **internal object name** (`StaticMeshActor_N`) in the same-named field —
  and additionally renames the array (`objects` vs `actors`).

This is *not* a request to redefine the inspect `name` to the label. The
internal object name is the **collision-safe key** the rest of the API depends
on (`actor.list` / `actor.get` emit it as a separate `name` field; the
`actor.*` resolver and other verbs accept it; display labels are explicitly
non-unique). Tests deliberately depend on `list_objects` filtering on `GetName()`
(`TestEnvironmentHandlers.cpp` renames spawned actors so the tag lands in the
object name). Swapping `name` to the label would break that contract and diverge
from `system.inspect.list_objects` / `actor.list`, where `name` is the internal
name. The real gap is that the read-only twins **omit the label entirely**, so a
human-facing audit must re-call the write twin (or reconcile paths) just to read
the name the user assigned.

The display label is plainly available (the write twin surfaces it from the same
actors), so the read-only path is strictly *less* useful for a human-facing audit
while being the documented "preferred" path for exactly that use case. An agent
asked to "report the actor names" — where the user means the labels they assigned
(`Checkpoint_A`/`B`/`C`) — gets back `StaticMeshActor_0/_4/_5` and must do a
separate name→label reconciliation (via the spawn responses' paths, or by
re-calling the write twin) to answer the question.

Compounding it, the `system.inspect` wiki overlay over-promises parity: it states
the inspect twins are *"same data, different intent labels"* as the `actor.*`
twins — which is **false** for the `name` field (internal name vs label). The
docs send an audit caller to the variant that hides the label *and* tell them it
is equivalent.

## Fix

Additive (non-breaking), mirroring the precedent set by `E-inspect-object-class-key-drift`
(which added the canonical `class` alongside the legacy `className` rather than
renaming) and by `actor.list` (which already emits BOTH `label`=`GetActorLabel`
and `name`=`GetName` per row):

1. Add a `label` field (the editor display label, `GetActorLabel`) to each
   `system.inspect` enumerator row — `system.inspect.find_by_tag`,
   `system.inspect.find_by_class`, `system.inspect.list_objects` — **alongside**
   the existing internal `name`. Do **NOT** redefine `name`: it stays the
   collision-safe internal object name (`GetName`). For `list_objects`, add
   `label` to the field-projection allow-list (kept by `namesOnly`, droppable via
   `fields`) so the projection contract matches `actor.list`. The `label` is
   output-only — it is **not** a filter input, preserving the existing
   name/class-substring filter parity.
2. Correct the docs: remove the false *"same data, different intent labels"*
   parity claim in `docs/wiki-src/system.inspect.md` and document that the
   `system.inspect.*` rows' `name` is the internal object name (`GetName`) while
   the `actor.*` twins put the display label in `name`; the inspect rows now also
   carry `label`.

## Evidence (live, ExampleProjectWelcome, identical world state)

Same query (`tag=PatrolCheckpoint`), three StaticMeshActors whose user-assigned
display labels are `Checkpoint_A` / `Checkpoint_C` / `Checkpoint_Spare`:

`system.inspect.find_by_tag {"tag":"PatrolCheckpoint"}` →
```json
{"objects":[
  {"name":"StaticMeshActor_0","path":".../PersistentLevel.StaticMeshActor_0","class":"StaticMeshActor"},
  {"name":"StaticMeshActor_4","path":".../PersistentLevel.StaticMeshActor_4","class":"StaticMeshActor"},
  {"name":"StaticMeshActor_5","path":".../PersistentLevel.StaticMeshActor_5","class":"StaticMeshActor"}],
 "count":3,"world":"auto","success":true}
```

`actor.find_by_tag {"tag":"PatrolCheckpoint"}` (same actors, same paths) →
```json
{"actors":[
  {"name":"Checkpoint_A","path":".../PersistentLevel.StaticMeshActor_0","class":"/Script/Engine.StaticMeshActor"},
  {"name":"Checkpoint_C","path":".../PersistentLevel.StaticMeshActor_4","class":"/Script/Engine.StaticMeshActor"},
  {"name":"Checkpoint_Spare","path":".../PersistentLevel.StaticMeshActor_5","class":"/Script/Engine.StaticMeshActor"}],
 "count":3,"world":"auto"}
```

The paths line up one-to-one (`StaticMeshActor_0` = `Checkpoint_A`, etc.), so the
two calls describe the same actors — but only the write twin exposes the label
the user reads. Originating task friction note, verbatim: *"Minor:
system.inspect.find_by_tag returns internal object names (StaticMeshActor_N), not
the user-facing display labels, so I mapped names→labels via the paths/GUIDs from
the actor.spawn responses to confirm membership — a small
discoverability/ergonomics gap, not a blocker."*

**Workaround:** Use `actor.find_by_tag` (not the read-only twin) when you need
display labels, or reconcile the `system.inspect.find_by_tag` paths against the
`actor.spawn` responses.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed during a tag-grouping audit task (seed `system.inspect.find_by_tag`). The docs pair `system.inspect.find_by_tag` with `actor.find_by_tag` as read-only/write counterparts and steer audit callers to the inspect variant, but on the identical query (`tag=PatrolCheckpoint`) the inspect twin returns `name:"StaticMeshActor_0/_4/_5"` (internal names, key `objects`) while `actor.find_by_tag` returns `name:"Checkpoint_A/_C/_Spare"` (display labels, key `actors`) for the same three actors (paths match one-to-one). The user asked to "report the actor names" meaning their assigned labels, so the recommended read-only path can't answer without a name→label reconciliation. Not a dup: `E-actor-list-no-class-filter` is about class-vs-name *filtering* on `actor.list`; `B-actor-find-by-class-short-name-fails` is the short-class-name resolution bug; neither covers the `name`=label-vs-internal-name asymmetry between the `system.inspect.*` queries and their `actor.*` twins. Proposed: report the display label (in `name` to match the twin, or as a new `label`/`displayName` field) on `system.inspect.find_by_tag` / `find_by_class` / `list_objects`, or at minimum document that their `name` is the internal object name while the `actor.*` twins return labels.
- `#2-reword` `OPEN` developer — Reworded to fix the mis-scoped headline remedy. Three validity lenses agreed the defect is real and present (`EnvironmentHandler.cpp:1554` GetName vs `QueryHandler.cpp:264` GetActorLabel), but the original lead option ("report the display label in `name`") is breaking: it contradicts the collision-safe-key invariant (`actor.list`/`actor.get` keep `name`=GetName; the `actor.*` resolver feeds on it) and the explicit test contract in `TestEnvironmentHandlers.cpp` that `list_objects` filters on `GetName()`. Retitled, rescoped severity-Low/ergonomic, and set the canonical **Fix:** to the ADDITIVE `label` field (mirroring the `E-inspect-object-class-key-drift` class/className additive precedent and `actor.list`'s existing label+name pair) plus the docs correction — explicitly NOT a `name` redefinition.
- `#3-fix` `IN-REVIEW` developer — Added an additive `label` field (`GetActorLabel`) alongside the existing internal `name` (`GetName`) on all three read-only enumerators in `Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp`: `system.inspect.find_by_tag`, `system.inspect.find_by_class`, and `system.inspect.list_objects` (label added to the field-projection allow-list, kept by `namesOnly`, output-only / not a filter input). `name` is unchanged (stays the collision-safe internal object name). Updated the three method summaries to document name=internal / label=display. Corrected the false "same data, different intent labels" parity claim in `docs/wiki-src/system.inspect.md` and the `list_objects` field-shape docs. Regression test `PinWright.system.inspect.find_by_tag.EmitsDisplayLabelKeepsInternalName` in `Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp` spawns a cube whose display label differs from its internal name, tags it, and asserts each of find_by_tag/find_by_class/list_objects returns the probe row with `label`==GetActorLabel AND `name`==GetName (name != label) — failing if the label field were dropped OR if `name` were redefined to the label.
