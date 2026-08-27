---
id: B-actor-set-folder-folderpath-no-folder-alias
title: "actor.set_folder requires 'folderPath' while its sibling actor.spawn_batch spells the identical concept 'folder' — and set_folder's own response emits 'folder', so one call uses both spellings"
status: OPEN
severity: Medium
category: bug
tags: [actor, actor-set-folder, actor-spawn-batch, param-naming, alias, outliner-folder, vocabulary-divergence, ergonomics]
encounters: 1
lastSeen: 2026-08-27T18:45:00+05:00
---

# The two `actor` verbs that take a World Outliner folder path disagree on its name, and `actor.set_folder` disagrees with itself between request and response

Assigning an actor to a World Outliner folder is done two ways, and they use two
different parameter names for the same string:

| verb | parameter | declared at |
|---|---|---|
| `actor.spawn_batch` | `folder` | `Handlers/Actor/SpawnBatchHandler.cpp:41` |
| `actor.set_folder`  | `folderPath` | `Handlers/Actor/ActorFolderHandler.cpp:23` |

Neither declares the other's spelling as an alias. Because the dispatcher
rejects undeclared payload keys, calling `actor.set_folder {folder: "..."}` —
the spelling the caller just used on `spawn_batch`, on the same folder, in the
same workflow — fails outright.

These are not distant verbs. They are the *pair* for this task: `spawn_batch`
sets the folder for actors you are creating, `set_folder` sets it for actors
that already exist. Anything spawned by a verb other than `spawn_batch` — every
`lighting.spawn_light`, `niagara.spawn_actor`, `environment.*` spawn — has to be
foldered with `set_folder` afterwards, so a build that keeps to an
Outliner-folder convention necessarily uses both spellings within minutes.

**The verb also contradicts itself inside a single call.** `actor.set_folder`
requires `folderPath` on input but emits `folder` on output, both at top level
and per row:

```
actor.set_folder {"actorName":"Portal_Light","folderPath":"Atlantis/Lighting"}
  -> {"folderPath":"Atlantis/Lighting",
      "updated":[{"name":"Portal_Light","label":"Portal_Light",
                  "objectName":"PointLight_0",
                  "path":"/Game/Maps/Atlantis.Atlantis:PersistentLevel.PointLight_0",
                  "folder":"Atlantis/Lighting"}],
      "updatedCount":1}
```

So the key a caller reads back from the result (`folder`) is the key that is
refused if fed to the next call.

## Which spelling is the outlier

Across the whole handler surface, `folder` is the majority spelling for a
"folder to put this in" parameter (4 declarations) and `folderPath` the
minority (2):

```
RPC_PARAM_*("folder", ...)
  Handlers/Actor/SpawnBatchHandler.cpp:41      World Outliner folder
  Handlers/Audio/AudioAnalysisHandler.cpp:1729 content folder
  Handlers/Editor/UtilityWidgetHandler.cpp:23  content-browser folder
  Handlers/UI/WidgetCreateHandler.cpp:29       content-browser folder

RPC_PARAM_*("folderPath", ...)
  Handlers/Actor/ActorFolderHandler.cpp:23     World Outliner folder
  Handlers/Asset/AssetDumpHandler.cpp:3083     content folder
```

Both concepts (Outliner folder, content folder) already appear under BOTH
spellings, so the divergence does not even encode a meaningful distinction —
it is not "`folderPath` means content path, `folder` means Outliner".

## Verbatim repro (replayed live via `mcp__pinwright__call`, UE 5.8)

```
actor.spawn_batch {"meshPath":"/Game/Atlantis/Meshes/SM_Temple_Podium",
                   "folder":"Atlantis/Temple",
                   "transforms":[{"location":{"x":0,"y":0,"z":0}}]}
  -> success, "folder":"Atlantis/Temple"

lighting.spawn_light {"lightType":"point","name":"Portal_Light",
                      "location":{"x":-300,"y":0,"z":1500}}
  -> success   (no folder parameter on this verb at all)

actor.set_folder {"actorName":"Portal_Light","folder":"Atlantis/Lighting"}
  -> [MISSING_REQUIRED_PARAM] Missing required parameter 'folderPath' (type: string)

actor.set_folder {"actorName":"Portal_Light","folderPath":"Atlantis/Lighting"}
  -> success
```

## Impact

One wasted round-trip per occurrence. This is a **loud** failure —
`MISSING_REQUIRED_PARAM` names the key it wants, so nothing is silently wrong
and no work is lost. Filed anyway because the conventions doc treats an
undiscoverable-but-working verb as a defect, and because the fix is one line
against a pairing every folder-convention build walks: this project's
`Docs/map/atlantis-spec.md` mandates that "every actor must land in one of"
ten named Outliner folders, so both verbs are on the main path for every agent
placing anything.

The error message is good as far as it goes but does not mention that the
sibling spawn verb uses a different word, which is the actual confusion.

## What it should do

Add `folder` as an accepted alias on `actor.set_folder`, keeping `folderPath`
working. The board's own convention requires each accepted alias to be its own
declared param — `agent-conventions.md`: "every accepted alias must be its own
`RPC_PARAM_OPT`; an undeclared alias read via `GetStringFirstOf` is dead code in
production" — and the codebase already has the helper for exactly this shape,
`ParamAliasUtils::MakeAliasParamSpec`, used a few lines away in
`Handlers/Actor/QueryHandler.cpp:50-51` to give `actor.list` both
`fields`/`field` and `namesOnly`/`names_only`. So:

```cpp
ParamAliasUtils::MakeAliasParamSpec(TEXT("folderPath"), TEXT("string"),
    TEXT("World Outliner folder path to assign, e.g. 'Prototype/Walls'. ..."),
    /*bRequired=*/true, TArray<FString>{TEXT("folderPath"), TEXT("folder")})
```

Optionally also echo `folder` alongside `folderPath` at top level (or rename the
echo) so the response and the request agree with each other.

## Distinct from related tickets

- `E-blueprint-param-name-path-vs-assetpath` (same family: two verbs, one
  concept, two parameter names) is the precedent for treating this class as
  actionable, but concerns the `blueprint` namespace's `path`/`assetPath`, a
  different pair.
- `B-create-physics-asset-skeletonpath-alias-dead` is about an alias that is
  *declared but non-functional*; here no alias exists at all.
- `B-actor-list-fields-unknown-key-silently-dropped` (filed same session) is the
  neighbouring `actor` vocabulary defect but the opposite failure mode — silent
  acceptance rather than loud refusal — and concerns values inside an array
  rather than a top-level key.

severity rationale: impact=loud failure costing one round-trip, no data loss, plus a self-inconsistent request/response vocabulary x reach=the standard spawn-then-folder pairing on any build that uses Outliner folders, and the only route for actors spawned by non-`spawn_batch` verbs -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Hit while placing the Atlantis central temple and its portal light. `actor.spawn_batch` had just taken `folder:"Atlantis/Temple"` successfully; `lighting.spawn_light` (which declares no folder parameter) then required a follow-up `actor.set_folder`, and the natural `folder:"Atlantis/Lighting"` was refused with `[MISSING_REQUIRED_PARAM] Missing required parameter 'folderPath'`. Retried with `folderPath` -> success. Confirmed from source that no alias exists: `ActorFolderHandler.cpp:23` declares `RPC_PARAM_REQ("folderPath", ...)` against `SpawnBatchHandler.cpp:41`'s `RPC_PARAM_OPT("folder", ...)`. Surveyed all folder-ish param declarations across `Handlers/` — `folder` 4, `folderPath` 2, with both spellings used for both Outliner and content folders, so the split carries no semantic meaning. Also recorded that `actor.set_folder`'s response emits `folder` per row while requiring `folderPath` on input. Proposed fix uses the in-repo `ParamAliasUtils::MakeAliasParamSpec` pattern already applied at `QueryHandler.cpp:50-51`. Classified TOOL BUG (discoverability/vocabulary; loud failure, hence Medium not High).
