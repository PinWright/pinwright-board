---
id: B-actor-list-fields-unknown-key-silently-dropped
title: "actor.list silently discards any fields[] key outside {label,name,path,class} — asking for the folder returns success with EMPTY row objects, and the wiki says the allow-list mirrors actor.describe (which does support folder)"
status: IN-REVIEW
severity: High
category: bug
tags: [actor, actor-list, fields, field-projection, silent-drop, allow-list, unknown-params, discoverability]
encounters: 1
lastSeen: 2026-08-27T18:45:00+05:00
---

# `actor.list` drops unrecognised `fields[]` entries without a word, and a projection made only of unrecognised keys returns rows that are empty JSON objects

`actor.list`'s `fields` parameter is a per-row allow-list. Its declared valid keys
are `label`, `name`, `path`, `class`. Any OTHER string in the array is accepted,
never matched, and never mentioned again: no `INVALID_ARGUMENT`, no `warnings[]`,
no echo of what was honoured. The response carries `count`, `totalMatches`,
`filter`, `matchMode` and `caseSensitive` — every one of which is echoed back —
but nothing about the projection that was actually applied.

Two failure grades follow from the same line of code:

1. **Partial projection** — `fields: ["name","folder","location"]` returns rows
   containing only `name`. The caller asked for three columns and got one, with
   no signal that two were discarded.
2. **Total projection wipe** — `fields: ["folder"]` (nothing but unrecognised
   keys) returns rows that are **empty JSON objects**. `count` and `totalMatches`
   still report the matched actors, so the call looks like a success that found
   the actor and then had nothing to say about it.

Grade 2 is the dangerous one: an empty `{}` row is indistinguishable from
"this actor has no such data", so a caller reasonably concludes the actors are
unfoldered rather than that the verb refused to answer.

## Why this is a contract break, not just a missing feature

- The **wiki explicitly promises the opposite**. `Saved/PinWright/wiki/actor.list.md:66`:
  "**`fields`** — allow-list `label`/`name`/`path`/`class` (array or bare string),
  such as `fields:["label","class"]`; **it mirrors `actor.describe`**."
  It does not mirror it. `actor.describe`'s `fields` allow-list is declared at
  `Handlers/Actor/DescribeHandler.cpp:20` as
  `name, label, path, class, level, folder, guid, tags, transform, properties, components`
  — `folder` among them. So the one sentence a caller reads to learn the key set
  names a sibling verb whose key set is a strict superset, and `folder` is exactly
  the key that exists on one and silently vanishes on the other.
- It contradicts the dispatcher's own house policy. `Dispatch/RpcDispatcher.cpp`
  rejects undeclared payload keys with `UNKNOWN_PARAMS` — verified live in this
  same session: `asset.list {path, recursive, limit}` returned
  `[UNKNOWN_PARAMS] Unknown parameter(s) for 'asset.list': [limit]. Valid parameters: [path, filter, recursive, pagination, depth, fields, namesOnly]`.
  So an unknown key at the top level is a loud, itemised refusal naming the valid
  set, while an unknown key one level down inside `fields[]` is silently eaten.

## Root cause (guilty source lines)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Actor/QueryHandler.cpp:71-85`:

```cpp
  const TSet<FString> Fields = Ctx.ReadFieldProjection(
      {TEXT("label"), TEXT("name"), TEXT("class")});
  const bool bProject = Fields.Num() > 0;
  const auto Wants = [&Fields, bProject](const TCHAR* Key) {
    return !bProject || Fields.Contains(FString(Key));
  };
  const bool bWantLabel = Wants(TEXT("label"));
  const bool bWantName  = Wants(TEXT("name"));
  const bool bWantPath  = Wants(TEXT("path"));
  const bool bWantClass = Wants(TEXT("class"));
```

`Ctx.ReadFieldProjection` lowercases and returns the caller's set verbatim; it
does not validate membership against the four keys the handler can actually
emit. The handler then probes the set only for those four. An entry like
`folder` therefore raises `Fields.Num()` — which is what flips `bProject` to
true and switches projection ON — while matching none of the four probes. When
every supplied key is unrecognised, `bProject` is `true` and all four
`bWant*` are `false`, so the row-building block below emits an object with no
fields set at all. The projection is turned on by the very key it cannot honour.

## Verbatim repro (replayed live via `mcp__pinwright__call`, UE 5.8, `/Game/Maps/Atlantis`)

Grade 2, the minimal case — one actor known to exist and known to be in folder
`Atlantis/Temple`:

```
actor.list {"filter":"Temple_Portal_Ring","fields":["folder"]}
  -> {"actors":[{}],"count":1,"totalMatches":1,"truncated":false,
      "world":"auto","worldPath":"/Game/Maps/Atlantis.Atlantis",
      "filter":"Temple_Portal_Ring","matchMode":"contains","caseSensitive":false}
```

One matched actor, one row, and the row is `{}`. For contrast, the same actor
through the sibling verb answers the question directly:

```
actor.describe {"actorName":"Temple_Portal_Ring"}
  -> ... "folder":"Atlantis/Temple" ...
```

Grade 1, how it was first hit:

```
actor.list {"fields":["name","folder","location"],"limit":0}
  -> {"actors":[{"name":"WorldSettings"},{"name":"Brush_0"}, ...]}   // name only
actor.list {"filter":"Temple_","matchMode":"contains","fields":["label","folder"],"limit":0}
  -> {"actors":[{"label":"Temple_Podium"}],"count":1,...}            // label only
```

## Impact

Cost in this session: two `actor.list` calls that answered nothing, then a
fallback to `actor.describe` purely to read one folder string — and
`actor.describe` returns the entire property and component tree for the actor
(~10 KB of JSON for a single `StaticMeshActor`, including the full
`FBodyInstance` twice) when all that was wanted was one field. On a
multi-agent build where "which folder is this actor in?" is the routine
question — the whole point of the Outliner-folder convention — the cheap verb
cannot answer it and the expensive verb floods the context.

`location` has the same shape: it is not in either allow-list, so there is no
projection that returns actor transforms from `actor.list` at all, and asking
for one is silent.

## What it should do

Either of these closes it; (a) is the minimal fix and matches house policy:

- **(a) Refuse unknown keys.** Validate the caller's set against the emittable
  keys and `SendError(INVALID_ARGUMENT, ...)` naming the offending entry AND the
  valid set — the same courtesy `UNKNOWN_PARAMS` already extends one level up.
  A projection that would emit nothing is never what the caller meant.
- **(b) Honour them.** Widen `actor.list`'s projection to the keys the wiki
  already claims it mirrors — at minimum `folder`, which `actor.describe`
  supports and which is cheap (`Actor->GetFolderPath()`), and `transform`.

Whichever is chosen, fix `Saved/PinWright/wiki/actor.list.md:66`'s "it mirrors
`actor.describe`" — under (a) that sentence is simply false, and it is the
sentence that caused this.

## Distinct from related tickets

- `F-actor-list-omits-location` (OPEN, Low, **feature**) asks for `location` /
  `transform` to be ADDED to this same allow-list, and explicitly frames itself
  as "an enrichment/convenience gap, NOT a documented-contract violation". This
  ticket is the complementary **bug**: not that a key is missing, but that
  supplying any key the handler cannot emit is swallowed without a word, and
  that a projection made entirely of such keys returns success-shaped rows that
  are empty objects. The two want different fixes — that one widens the
  allow-list, this one demands the allow-list be *enforced* (or the wiki's
  "mirrors `actor.describe`" claim be made true). Fixing `F-actor-list-omits-location`
  alone would silence the `location` case while leaving `folder` and every
  typo (`Label`, `clas`, `pathname`) silently empty.
- `E-actor-list-no-limit-spills` (IN-REVIEW) shipped the `fields` projection
  this ticket reports on, deliberately scoped to `label/name/path/class`. The
  silent-drop behaviour is a property of how that projection reads its input,
  not a regression of it.
- `E-blueprint-param-name-path-vs-assetpath` is the same *family* of
  vocabulary friction one namespace over, but concerns a top-level parameter
  name (which the dispatcher rejects loudly), not values inside an array
  (which nothing checks).

severity rationale: impact=silent no-op returning a success-shaped response with empty rows, indistinguishable from "no such data", against a documented mirror promise x reach=`actor.list` is the primary world-inspection verb and folder is the routine question on any multi-agent build -> High

## History
- `#1-initial-repro` `OPEN` reporter — Hit while placing the Atlantis central temple: needed to confirm which actors already sat in `Atlantis/Temple` / `Atlantis/Rubble` / `Atlantis/Lighting` before spawning, to avoid duplicating a tier after an editor crash. `actor.list {fields:["name","folder","location"]}` returned name-only rows with no warning; `fields:["label","folder"]` returned label-only. Narrowed live to the minimal case `fields:["folder"]` -> `{"actors":[{}],"count":1}` — projection switched ON by a key the handler cannot emit, so every `bWant*` is false and the row is empty. Root cause read from source: `QueryHandler.cpp:71-85`, `Ctx.ReadFieldProjection` returns the caller's set unvalidated and `Wants()` probes only `label|name|path|class`. Cross-checked `DescribeHandler.cpp:20`, whose allow-list does include `folder`, against `wiki/actor.list.md:66` "it mirrors `actor.describe`" — the doc promise is false. Contrast case proving house policy is loud refusal: `asset.list {limit:200}` -> `UNKNOWN_PARAMS` itemising the valid set. Classified TOOL BUG (silent no-op on a normal path).
- `#2-reject-unknown-fields-and-emit-folder` `IN-REVIEW` developer — Both halves of the ticket, in `Handlers/Actor/QueryHandler.cpp` (`actor.list` only). (1) Added an `EmittableFields` set {label, name, path, class, folder} and a validation pass immediately after `Ctx.ReadFieldProjection`: any key not in it is now `SendError(ErrorCodes::ERR_INVALID_PARAMS, ...)` naming the offending entries and the valid set, matching the dispatcher's UNKNOWN_PARAMS message shape (no new ERR_* constant; `ERR_INVALID_PARAMS` is already the house code for nested unknown-key rejection, cf. `ImageOps::RejectUnknownKeys`). Both grades fail now: `fields:["folder"]` (all-unknown, formerly `{}` rows) and `fields:["name","location"]` (partial drop). (2) `folder` is now emittable via `Actor->GetFolderPath()`, opt-in only — it is NOT added to the default row or to `namesOnly`, so an unprojected `actor.list` keeps its prior four-column shape and the verb does not get wider on a level where it already spills. Param help and the method summary updated to state the valid set, the opt-in folder, and that unknown keys are refused. New test file `Tests/World/TestActorListFieldProjection.cpp`: `PinWright.actor.list.FieldProjection.UnknownFieldIsRejected` (world-independent; asserts INVALID_PARAMS + offending key + valid set for all-unknown, mixed and typo cases, and that an all-emittable projection still succeeds) and `PinWright.actor.list.FieldProjection.FolderIsProjectable` (spawns a non-transient PointLight probe under `FScopedEditorWorldActorGuard`, assigns a synthetic folder via `actor.set_folder`, asserts `fields:["folder"]` returns a non-empty row whose folder equals `AActor::GetFolderPath`, that `fields:["name","folder"]` returns both columns and drops path, and that the unprojected row does NOT carry folder). Not compiled or run — the orchestrator owns builds. Wiki correction for `docs/wiki-src/actor.md` handed to the orchestrator, not edited here.
