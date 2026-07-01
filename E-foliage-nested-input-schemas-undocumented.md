---
id: E-foliage-nested-input-schemas-undocumented
title: "foliage.add_instances / foliage.create_procedural document param names but not the nested object shapes (location/rotation/scale, bounds.location+size, foliageTypes[].meshPath+density), forcing a read of FoliageHandler.cpp to author the payload"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, foliage, wiki, schema, nested-object, discovery, add_instances, create_procedural]
---

# foliage write verbs name their params but not the nested field shapes

`foliage.add_instances` and `foliage.create_procedural` are the two foliage
write verbs whose payloads are *nested objects/arrays* rather than flat
scalars. Their wiki/registry entries document the top-level param **names** and
a one-line gloss, but stop exactly where the structure begins — so an agent
authoring the call has the param name but not the field layout it must fill in,
and has to fall back to reading the plugin C++ to recover the schema.

The wiki-generated pages (verbatim) say:

- `foliage.add_instances`:
  - `transforms` (`array`, optional): "Array of {location, rotation, scale} transforms"
  - `locations` (`array`, optional): "Array of {x,y,z} positions (legacy, default rotation/scale)"
- `foliage.create_procedural`:
  - `bounds` (`object`, required): "Volume bounds with location and size"
  - `foliageTypes` (`array`, required): "Array of foliage type configs with meshPath and density"

Those glosses name the *keys* but never the *value shapes*, which is the part
that actually trips authoring. From `FoliageHandler.cpp` the real schema is
(line numbers re-anchored to current source):

- **`transforms[]`** (`:654-728`): each entry is
  `{ location, rotation, scale }` where
  - `location` is `{x,y,z}` **or** `[x,y,z]` (`:668-684`) — required; an entry
    with no valid location is silently skipped (`:682 continue`).
  - `rotation` is `{pitch,yaw,roll}` **or** `[pitch,yaw,roll]` (`:687-703`) — optional.
  - `scale` is `{x,y,z}` **or** `[x,y,z]` **or** a scalar `uniformScale`
    (`:705-725`, `uniformScale` at `:721`) — optional. None of these three
    accepted forms, nor the `uniformScale` alias, nor the "location is mandatory
    / silently dropped" rule, appear in the docs.
- **`bounds`** (`:853-882`) is `{ location:{x,y,z}, size:{x,y,z} }`, where `size`
  may alternatively be `[x,y,z]` (`:876-882`). The `{x,y,z}` decomposition of
  `location`/`size` and the array form of `size` are undocumented.
- **`foliageTypes[]`** (`:918-924`) is `{ meshPath, density }` per entry —
  and **only** those two fields are read (no per-entry `minScale`/`maxScale`/
  `alignToNormal`, unlike `foliage.add_type`, which DOES accept those at its
  registration `:464-467`). The docs say "configs with meshPath and density"
  but don't make clear that those are the *only* honored per-entry keys, so an
  agent may reasonably (and wrongly) try to pass scale/align overrides per type.

This is a *discovery* gap, not a bug: every call in this task succeeded. But the
cost was a detour into the handler source to author two of the six calls.

## Why it matters — the process friction (this task)

Seed story: a vegetation-dressing pass that created two foliage types, painted a
grass patch, added rock instances via **full transforms** (rotations +
non-uniform scales), queried instances, and set up a procedural foliage volume.
9 calls, all `ok` / no errors. The friction note (verbatim):

> "Mostly smooth (one upfront wiki read covered all params), but two real
> ergonomic gaps: the wiki for add_instances/create_procedural never documents
> the nested transforms/bounds/foliageTypes object shapes, so I had to read the
> plugin C++ (FoliageHandler.cpp) to confirm the {location,rotation,scale} and
> {location,size}+{meshPath,density} schemas; ..."

So the up-front wiki read (call #1, the only navigation step) covered the *flat*
params fine, but the two **nested-payload** verbs each forced a source dive to
confirm the inner object shape before the agent would commit the call. That is
exactly the "repeated/extra discovery before the right call" friction shape —
here resolved not by a retry loop (the agent was careful) but by leaving the
tool surface entirely to read C++.

This is **distinct from** the judge-filed `E-foliage-get-instances-drops-scale`,
which is about the *read-back / output* asymmetry of `foliage.get_instances`.
This ticket is about the *input / authoring* discoverability of the two nested
write verbs — different methods, different direction (write vs read), different
remedy. They touch the same overlay file: that sibling's fix already added a
`### foliage.get_instances` H3 section to `docs/wiki-src/foliage.md` (so the
per-method H3 authoring pattern already exists there), but there is still **no
authoring section for `add_instances` or `create_procedural`** — this ticket
adds those two, building on the existing structure rather than treating the file
as empty.

## What it should do / how to fix

Docs-only (downstream wiki process — not a code change): the overlay
`docs/wiki-src/foliage.md` should add per-method authoring blocks for
`foliage.add_instances` and `foliage.create_procedural` that spell out the
nested shapes with a tiny example each, e.g.

```jsonc
// add_instances
{ "foliageTypePath": "/Game/Foliage/Rocks",
  "transforms": [
    { "location": {"x":0,"y":0,"z":0},
      "rotation": {"pitch":0,"yaw":45,"roll":0},   // optional
      "scale":    {"x":1,"y":1,"z":2} } ] }          // or [x,y,z] or "uniformScale": 2

// create_procedural
{ "name": "MeadowSpawner",
  "bounds": { "location": {"x":0,"y":0,"z":0}, "size": {"x":2000,"y":2000,"z":500} },
  "foliageTypes": [ { "meshPath": "/Engine/BasicShapes/Cube", "density": 300 } ],
  "seed": 12345 }
```

and note the two non-obvious rules: a `transforms` entry **without a valid
location is silently dropped**, and `foliageTypes[]` honors **only** `meshPath`
+ `density` (no per-type scale/align). That converts both verbs from
"read the handler to find the shape" to "copy the documented example."

**Workaround:** until the overlay is filled in, the nested shapes are recoverable
only from `FoliageHandler.cpp` (`transforms` `:654-728`, `bounds` `:853-882`,
`foliageTypes` `:918-924`).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of a foliage
  vegetation-dressing task: 9 calls, all `ok`, no retries/errors, but the
  friction note flags that `foliage.add_instances` / `foliage.create_procedural`
  document param names only, forcing a read of `FoliageHandler.cpp` to confirm
  the nested `{location,rotation,scale}` (transforms), `{location,size}` (bounds),
  and `{meshPath,density}` (foliageTypes) shapes before authoring. Confirmed
  against the live wiki-generated pages (`foliage.add_instances.md`,
  `foliage.create_procedural.md`): both stop at the param name + one-line gloss
  and never give the inner field layout; the overlay `docs/wiki-src/foliage.md`
  is a 6-line namespace blurb with no per-method docs. Handler-confirmed schema:
  transforms accept location `{x,y,z}`|`[x,y,z]` (required, else silently skipped
  `:668`), rotation `{pitch,yaw,roll}`|`[..]` (opt), scale `{x,y,z}`|`[..]`|
  `uniformScale` (opt) (`:655-708`); bounds = `{location:{x,y,z}, size:{x,y,z}|[..]}`
  (`:840-866`); each foliageTypes entry reads ONLY `meshPath`+`density`
  (`:904-910`). Distinct PROCESS angle from the judge-filed
  `E-foliage-get-instances-drops-scale` (that is read-back/output of
  get_instances; this is input/authoring discoverability of the two nested write
  verbs). Proposed deliverable: per-method authoring blocks with nested examples
  + the silent-drop and meshPath/density-only caveats in `docs/wiki-src/foliage.md`.
  Dedup: ripgrep across OPEN/closed found no foliage `add_instances` /
  `create_procedural` input-schema docs ticket; the only foliage tickets are the
  get_instances read-back one (different direction) and PCG/perf mentions
  (different methods).
- `#2-reword-and-add-overlay-sections` `IN-REVIEW` developer — REWORD then
  implement. Reword: corrected two currency drifts the validity lenses flagged —
  (a) the body/Workaround line-number citations were re-anchored to current
  `FoliageHandler.cpp` (transforms `:654-728`, silent-drop `continue` `:682`,
  scale/`uniformScale` `:705-725`/`:721`, bounds `:853-882`, foliageTypes
  `:918-924`, plus `add_type`'s contrasting `:464-467`); (b) the stale claim that
  `docs/wiki-src/foliage.md` is "a 6-line namespace blurb with no per-method docs
  at all" was corrected — the sibling `E-foliage-get-instances-drops-scale` fix
  already added a `### foliage.get_instances` H3 there, so this ticket now builds
  on the existing per-method structure. Implement (docs-only, the established
  overlay pattern): added `### foliage.add_instances` and
  `### foliage.create_procedural` H3 sections to
  `Docs/wiki-src/foliage.md` documenting the nested shapes —
  transforms[] `{location(required, silently dropped if absent), rotation,
  scale}` with the `uniformScale` scalar alias and the legacy `locations`
  fallback; bounds `{location, size}` (size object|array); foliageTypes[] honors
  ONLY `meshPath`+`density` (asymmetry vs `foliage.add_type`) — each with a JSON
  example. Regression test:
  `Source/PinWright/Private/Tests/Infra/TestFoliageNestedInputSchemaDocs.cpp`
  (two `IMPLEMENT_SIMPLE_AUTOMATION_TEST` cases) renders the live
  `foliage.add_instances` / `foliage.create_procedural` method pages through
  `WikiHandler::RenderPage` (production render path, not a copy) and asserts the
  overlay-exclusive markers (silent-drop rule, `uniformScale`, `locations`
  fallback, meshPath/density-only + `add_type` contrast) survive — reverting the
  overlay makes `LoadMethodSection` return empty and these fail. No handler/code
  behavior changed.
- `#3-friction-not-reproduced` `OPEN` reporter — Counter-evidence on
  reproducibility/severity from an independent scatter task (seed `ScatterRocks`
  from `/Game/ExampleContent/Landscapes/Meshes/SM_Rock`, 6 instances mixing 4
  object-form scales `{x,y,z}` + 2 `uniformScale` scalars, distinct loc/yaw/scale
  each). This agent did **not** dive into `FoliageHandler.cpp` — the friction note
  (verbatim): "the foliage.add_instances wiki page documented the nested transform
  shape (location/rotation/scale, uniformScale as a sibling key, silent-drop
  warning) clearly enough that every call worked first try." So `add_instances`
  was authored correctly on the first attempt (object-form *and* `uniformScale`
  forms both used) purely from the wiki, with no source read. Confirms the gap is
  **intermittent, not a hard blocker**: the registry gloss is unchanged
  (`FoliageHandler.cpp:599` still "Array of {location, rotation, scale}
  transforms" — param-name-only) and the overlay `docs/wiki-src/foliage.md` is
  still the 6-line namespace blurb with no per-method docs, yet a careful agent
  recovered the nested shape (incl. the `uniformScale` sibling key) without
  leaving the tool surface. Supports keeping this Low and treating the overlay
  authoring blocks as a discoverability *smoothing* (some agents dive, some don't)
  rather than a correctness fix. Note this task only exercised `add_instances`
  (not `create_procedural`), so the `bounds`/`foliageTypes` half of the ticket is
  untouched by this evidence. No new file — appended to this OPEN ticket.
