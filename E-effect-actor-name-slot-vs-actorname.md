---
id: E-effect-actor-name-slot-vs-actorname
title: "actor.get_transform / set_transform / get_bounding_box / get declare a bare actorName slot with ZERO aliases — they reject the actorPath/objectPath key that spawn/duplicate return, lagging the actor.describe/get_components alias precedent"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [actor, param-alias, actorname, actorpath, objectpath, get_transform, set_transform, bounding_box, drift]
encounters: 2
lastSeen: 2026-07-02T09:34:50.7138314+03:00
---

# The `actor.*` transform/readback verbs carry NO actor-identity aliases — `actorPath`/`objectPath` hard-fail `MISSING_REQUIRED_PARAM`

> **Reword note (2026-06-23):** the original ticket framed this as an
> `effect.*`-vs-`actor.*` cross-namespace `name` flip and asked to (a) add an
> `actorName` alias to `effect.spawn_niagara` and (b) add a `name` alias to the
> `actor.*` transform verbs. Half (a) is **already shipped** — `effect.spawn_niagara`
> declares `RPC_PARAM_OPT("name", "string", "Actor label (also accepts actorName)")`
> (`Handlers/VFX/EffectHandler.cpp:1090`) and its body reads `name` then falls back
> to `actorName` (`:1186`/`:1188`). Half (b) — adding the bare key `name` — was
> **dropped**: `name` is already a *distinct* canonical key meaning "search
> substring" on `actor.find_by_name` (`QueryHandler.cpp:167`,
> `"Substring to match against label/name/path"`), and `actor.spawn`'s slot is
> `actorName`, not `name` (`SpawnHandler.cpp`). The shared actor-identity alias set
> `ActorNameParamUtils::ActorNameKeys()` is *deliberately* path-shaped
> (`{actorName, objectPath, actorPath, actor_name}` — NOT `name`/`label`), matching
> the DONE precedent `E-material-editor-param-name-drift #2` that left a polysemous
> key unaliased on purpose. So the real, precedent-aligned defect is below.

`E-actor-verbs-reject-actorpath-slot` (DONE/IN-REVIEW) migrated `actor.describe`
and `actor.get_components` to `ActorNameParamUtils::ActorNameParamReq()`, giving
their `actorName` slot the `{objectPath, actorPath, actor_name}` aliases — the
keys `actor.spawn` / `actor.spawn_instance` / `actor.duplicate` hand the spawned
actor back under (`actorPath`), plus the path-shaped `objectPath` declared as an
INPUT slot in `volume.*` / `world.*` / `sequencer.*`. But the transform/readback
verbs were **never migrated** and still declare the bare canonical key with
**zero aliases**:

- `actor.get_transform` (`Handlers/Actor/ActorTransformHandler.cpp:92`):
  `RPC_PARAM_REQ("actorName", ...)`, read via `Ctx.GetString(TEXT("actorName"))`
  (:95) — no aliases at all.
- `actor.set_transform` (`:29`, read `:35`) — same bare slot.
- `actor.get_bounding_box` (`:129`, read `:132`) — same bare slot.
- `actor.get` (`Handlers/Actor/QueryHandler.cpp:128`, read `:131`) — same bare slot.
- `actor.get_metadata` (`Handlers/Actor/QueryHandler.cpp:358`, read `:361`) — same bare slot (lighter sibling of `actor.get`).

The dispatcher's `ValidateHandlerParams` rejects any payload missing the literal
`actorName` (or a declared alias) **before the body runs**, so a caller who
reuses the `actorPath`/`objectPath` key that spawn/duplicate return hard-fails
`[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName'` on the very
first readback — even though `McpActorUtils::FindActorByName` already resolves a
path value (it matches label OR internal name OR `GetPathName`). These four
verbs lag behind the alias coverage their sibling readers (`actor.describe`,
`actor.get_components`) already got.

CLAUDE.md's "camelCase and snake_case aliases" rule does not cover this —
`actorPath`/`objectPath` are distinct path-shaped names, not casing variants of
`actorName` — so reusing the spawn-return key is a hard error, not a silent
accept.

## Repro

The friction surfaced as a `name`-vs-`actorName` retry at the readback step
after an `effect.*` Niagara workflow (`actor.get_transform {name:"FountainPreview"}`
→ `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName'`, corrected
with `{actorName:...}`). The `name` spelling is intentionally NOT accepted (it
means "search substring" on `actor.find_by_name`), but the same dispatcher
pre-check that rejects `name` *also* rejects the path-shaped keys spawn/duplicate
return — which IS the real, fixable gap:

1. `actor.spawn {classPath:..., actorName:"Foo"}` → returns `{actorPath:"/Game/.../Foo"}`.
2. `actor.get_transform {actorPath:"/Game/.../Foo"}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName' (type: string)`
   — the spec carries zero aliases, so `ValidateHandlerParams` rejects the
   `actorPath` key spawn just handed the caller, before the body (which would
   resolve a path via `FindActorByName`) ever runs.

Sibling readers `actor.describe` / `actor.get_components` already accept
`actorPath`/`objectPath` (via `E-actor-verbs-reject-actorpath-slot`); the
transform verbs and `actor.get` do not. The error is accurate but the round-trip
is pure drift overhead, exactly the shape of the precedent alias tickets.

## What it should do

Migrate the un-aliased `actor.*` reader/transform verbs to the existing
`ActorNameParamUtils` helper (the same change `E-actor-verbs-reject-actorpath-slot`
applied to `actor.describe` / `actor.get_components`), so they inherit the
deliberate path-shaped alias set `{actorName, objectPath, actorPath, actor_name}`:

- **`actor.get_transform`** (`ActorTransformHandler.cpp:90-93`),
  **`actor.set_transform`** (`:27-33`), **`actor.get_bounding_box`** (`:127-130`),
  **`actor.get`** (`QueryHandler.cpp:126-129`), and **`actor.get_metadata`**
  (`QueryHandler.cpp:356-359`): replace the bare
  `RPC_PARAM_REQ("actorName", ...)` spec with
  `ActorNameParamUtils::ActorNameParamReq(...)` and read the value body-side via
  `ActorNameParamUtils::RequireActorName(Ctx, Out)` (or `ResolveActorName`) so
  `actorPath`/`objectPath`/`actor_name` resolve end-to-end.
- **Do NOT** add `name` or `label` to the shared alias set — `name` is the
  search-substring canonical key on `actor.find_by_name`, and the set is kept
  path-shaped by design (precedent `E-material-editor-param-name-drift #2` left a
  polysemous key unaliased on purpose). The residual `name`-vs-`actorName` retry
  is closed by the docs half below, not by a dispatcher alias.

## Docs angle (`docs/wiki-src/actor.md`)

`docs/wiki-src/actor.md` has per-method H3 sections but **none for
`actor.get_transform`** (`### actor.describe`, `### actor.get_components`, etc.
exist; the transform readback verb and `actor.get` have no section). An
`### actor.get_transform` H3 naming the `actorName` slot — and noting it accepts
the `actorPath`/`objectPath` aliases that `actor.spawn` returns but NOT the bare
`name` (which is the search key on `actor.find_by_name`) — closes the residual
`name`-vs-`actorName` discovery friction the dispatcher alias deliberately does
not. The edit itself is the downstream wiki process, not this ticket.

## Not a duplicate of

- `B-set-niagara-param-no-validation` (judge-filed this task) — that is the
  `effect.set_niagara_parameter` silent `applied:true` OUTCOME bug; this is the
  PROCESS param-name flip on the readback step.
- `E-geometry-create-name-vs-actorname` / `E-volume-create-name-vs-volumename`
  — same family, different namespaces; neither touches `effect.*` or the
  `actor.get_transform` `actorName`-only slot. This one is the first to flag the
  generic `actor.*` readback as the missing-`name`-alias side of the drift.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the magic-fountain
  Niagara runtime-preview task (seed `effect.set_niagara_parameter`; outcome
  `tool_bug`, judge filed `B-set-niagara-param-no-validation` for the
  silent-success defect). Distinct PROCESS angle: a single misuse-then-correct
  round-trip on the final readback — `actor.get_transform {name:"FountainPreview"}`
  → `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName'`, corrected
  on retry with `actorName`. Root cause is cross-namespace actor-name drift: the
  8-call `effect.*` workflow addresses the actor as `name` (`spawn_niagara`,
  `EffectHandler.cpp:1184`) then `systemName` (operate verbs `:764`/`:810`/`:1058`,
  three of which already alias `actorName`), but `actor.get_transform`
  (`ActorTransformHandler.cpp:92`) is `actorName`-only with no `name` alias
  despite its "Display label or name" description. New namespace pair, not
  covered by `E-geometry-create-name-vs-actorname` (OPEN),
  `E-volume-create-name-vs-volumename` (OPEN), or the DONE path/param-alias
  precedents. Fix: dispatcher `FParamSpec` alias from
  `E-blueprint-param-name-path-vs-assetpath #4` — add `actorName` to
  `effect.spawn_niagara`'s `name` slot and `name` to the `actor.*` transform
  verbs' `actorName` slot; plus an `effect.md` / `actor.md` wiki note (no
  `### actor.get_transform` section exists today).
- `#2-more-evidence-actor-describe` `OPEN` reporter — Cross-task aggregation: the
  evening-blockout greybox struggle audit (namespace `environment.build`; outcome
  `nonrepro`; judge tracked the `export_snapshot` stub under
  `B-export-snapshot-empty-stub`) hit the same `name`→`actorName` flip on a
  *different* `actor.*` reader, `actor.describe`. After
  `environment.build.create_sky_sphere` the agent ran `actor.describe {name:"SkySphere"}`
  (reusing the create-side `name` spelling) →
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName' (type: string)`,
  corrected on retry with `{actorName:"SkySphere"}` (success). One wasted
  `is_error` call, zero blocked progress — same guessability/round-trip overhead
  as the `actor.get_transform` repro above. Friction note (verbatim): *"actor.describe
  rejected my first param name (used 'name' instead of 'actorName')."* Confirms the
  missing-`name`-alias gap spans the whole `actor.*` reader cluster
  (`actor.describe` here, `actor.get_transform` in `#1`), so the dispatcher
  `FParamSpec` `name`-alias fix should cover `actor.describe`'s `actorName` slot
  (`DescribeHandler.cpp:12,19`) too, not just the transform verbs.
- `#3-actor-spawn-label-self-contradicts` `OPEN` reporter — Cross-task
  aggregation, new spelling + new verb in this family: the cinematic
  establishing-shot struggle audit (namespace `misc`; outcome `ergo`; judge
  filed `E-create-camera-cine-doc` for the cine-doc gap). On the **spawn verb
  itself** the agent typed `actor.spawn {classPath:"CineCameraActor", label:"…"}`
  → `[UNKNOWN_PARAMS] Unknown parameter(s) for 'actor.spawn': [label]. Valid
  parameters: [classPath, actorName, meshPath, location, rotation].`, corrected
  on retry with `{actorName:"HeroShotCineCamera"}` (success). One wasted
  `is_error` call, zero blocked progress — same guessability round-trip as the
  reader-verb repros, but it extends the family two ways the prior history
  entries don't cover: (a) the rejected spelling here is **`label`** (not
  `name`), and (b) the verb is the **`actor.spawn` producer**, not a downstream
  reader. The self-contradiction makes `label` an especially natural wrong
  guess: `actor.spawn`'s `actorName` param help literally reads **"Display
  label for the spawned actor"** (`SpawnHandler.cpp:56`) — the help calls the
  value a "Display label" while the key is `actorName` and `label` is rejected.
  So beyond adding `name` to the alias set, the `FParamSpec` fix should also
  add **`label`** as an alias on `actor.spawn`'s `actorName` slot (and ideally
  the other actor verbs whose help all say "Display label or name"), OR the
  param help should stop calling the value a "Display label" if `label` is not
  accepted. Friction note (verbatim): *"two UNKNOWN_PARAMS retries (cine, label)
  before the error messages gave the right slot names — those errors were clear
  and self-correcting."* (The `cine` half of that note is `E-create-camera-cine-doc`'s;
  the `label` half is this family's.) Docs angle unchanged: `docs/wiki-src/actor.md`
  still has no `### actor.spawn` section naming the `actorName` slot — adding one
  that says "name the spawned actor with `actorName` (also accepts `label`/`name`)"
  closes the discovery half.
- `#4-reword-and-fix` `IN-REVIEW` developer — Reworded to the real, precedent-aligned
  defect and implemented. Two halves of the original ask were dropped: (a) the
  `effect.spawn_niagara` `actorName` alias is **already shipped**
  (`Handlers/VFX/EffectHandler.cpp:1090` declares `RPC_PARAM_OPT("name", "string",
  "Actor label (also accepts actorName)")`, body reads `name` then `actorName` at
  `:1186`/`:1188`), and (b) adding the bare key `name`/`label` to the actor-identity
  alias set was rejected — `name` is a *distinct* canonical key = "search substring"
  on `actor.find_by_name` (`QueryHandler.cpp:167`) and the shared
  `ActorNameParamUtils::ActorNameKeys()` is deliberately path-shaped
  (`{actorName, objectPath, actorPath, actor_name}`, mirroring the DONE
  `E-material-editor-param-name-drift #2` decision to leave a polysemous key
  unaliased). **Fix:** migrated the five un-aliased `actor.*` reader/transform verbs
  to `ActorNameParamUtils::ActorNameParamReq()` + `RequireActorName(Ctx, Out)` —
  `actor.set_transform`, `actor.get_transform`, `actor.get_bounding_box`
  (`Handlers/Actor/ActorTransformHandler.cpp`) and `actor.get` + `actor.get_metadata`
  (`Handlers/Actor/QueryHandler.cpp`) — so the `actorPath`/`objectPath`/`actor_name`
  keys that `actor.spawn`/`duplicate` return now resolve at the wire level instead of
  hard-failing `MISSING_REQUIRED_PARAM 'actorName'`. This brings them up to the
  `actor.describe`/`actor.get_components` precedent (`E-actor-verbs-reject-actorpath-slot`).
  **Test:** extended `Tests/World/TestActorNameParamAlias.cpp` — added the five
  verbs to the `ActorReaderVerbs()` list driving both the static spec-alias check
  (`FActorReaderVerbsDeclareActorPathAliasTest`) and the end-to-end wire-acceptance
  check (`FActorReaderVerbsAcceptActorPathAliasOnWireTest`); reverting the migration
  fails both (Aliases empty → static fail; dispatcher rejects `{actorPath:...}` →
  wire fail). Docs half (an `### actor.get_transform` H3 on `actor.md`) is downstream
  wiki work, not this ticket.
- `#5-evidence-actor-key-spelling` `IN-REVIEW` reporter — Cross-task aggregation, a
  THIRD wrong spelling for this family: the torch-lit-dungeon preview-lighting
  struggle audit (namespace `effect`; outcome `clean`; no judge filing). At the
  pulse-verification readback the agent typed `actor.describe {actor:"TorchLight_A"}`
  → `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName' (type: string).
  Accepted aliases: actorName, objectPath, actorPath, actor_name`, corrected on
  retry with `{actorName:"TorchLight_A"}` (success). One wasted `is_error` call,
  zero blocked progress — same single round-trip as `#1`/`#2`/`#3`. New facts vs
  prior history: (a) the rejected key here is the bare **`actor`** (the namespace
  noun itself) — not `name` (`#1`/`#2`) or `label` (`#3`); `actor` is an especially
  natural shorthand and is *not* in the deliberately path-shaped alias set, so it
  correctly hard-fails. (b) Unlike the `#4` migration target list
  (`get_transform`/`set_transform`/`get_bounding_box`/`get`/`get_metadata`), this
  hits **`actor.describe`** — which *already* carries the path-shaped aliases
  (`E-actor-verbs-reject-actorpath-slot`), confirming the residual friction is the
  missing **human-shorthand** alias (`actor`/`name`/`label`), NOT the path-shaped
  set, and is closeable only by the docs half (the `### actor.describe` H3 / a
  per-verb note that the slot is `actorName`, not the bare `actor`/`name`). The
  error message here is exemplary — it lists every accepted alias inline — which is
  why this stayed a clean one-retry; the friction is purely first-guess
  discoverability. Friction note (verbatim): *"two parameter-name guesses were
  rejected (actor.list wants 'filter' not 'nameFilter'; actor.describe wants
  'actorName' not 'actor') but the error messages listed the valid keys so each was
  a single clean retry."* (The `actor.list` half of that note is tracked on
  `E-actor-list-no-class-filter #4`.)
- `#6-additional-spawn-name-generic-guess` `IN-REVIEW` reporter — Cross-task
  aggregation, the generic **`name`** spelling now confirmed rejected on the
  **`actor.spawn` producer itself** (prior actor.spawn evidence in `#3` used
  `label`; the `name` spelling was previously only seen on reader verbs in
  `#1`/`#2`). Struggle audit of the candelabra scene-dressing task (focus
  `actor.duplicate_component`; outcome `done`; judge filed
  `B-duplicate-component-clones-editor-sprite` for the light-billboard clone bug —
  unrelated to this family). The agent's first spawn typed
  `actor.spawn {meshPath:"/Engine/BasicShapes/Cylinder", name:"CandelabraBase"}` →
  `[UNKNOWN_PARAMS] Unknown parameter(s) for 'actor.spawn': [name]. Valid
  parameters: [classPath, actorName, meshPath, location, rotation, scale].`,
  corrected on retry with `{actorName:"CandelabraBase"}` (success). One wasted
  `is_error` call, zero blocked progress — same single self-correcting round-trip
  as `#1`–`#5`; the error enumerates the valid params so recovery is one hop. New
  fact vs `#3`/`#4`: the `#4` reword dropped bare `name` from the SHARED reader
  alias set because `name` collides with `actor.find_by_name`'s "search substring"
  canonical key — but that collision does **not** exist on the *producer*
  `actor.spawn` (no `name`-means-search slot there), so a **spawn-local** `name`
  (and `label`, per `#3`) alias on `actor.spawn`'s `actorName` slot closes this
  producer-side first-guess friction WITHOUT reopening the shared-reader-set
  polysemy concern `#4` correctly avoided. Docs half unchanged and still open:
  `docs/wiki-src/actor.md` has no `### actor.spawn` section naming the `actorName`
  slot (also targeted by `E-spawn-no-scale-param #2`, which added that section —
  coordinate, don't duplicate). Friction note (verbatim): *"Minor: one retry on
  actor.spawn (I passed 'name'; the error listed valid params so I switched to
  'actorName')."*
