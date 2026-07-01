---
id: E-material-create-combined-assetpath-split
title: "material.authoring.create_material wants the asset path pre-split into name+path; agents pass one combined assetPath (the brief's natural full path) and eat a MISSING_REQUIRED_PARAM 'name' round-trip"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [material, param-alias, create_material, name, path, assetpath, split, drift, docs]
claimedBy: fuzz2
claimedAt: 2026-06-24T07:13:47.4354999+03:00
---

# `material.authoring.create_material` requires `name`+`path` split; the natural "one full asset path" guess (`assetPath`) hard-fails

Same param-name-guessability family as `E-level-create-name-path-alias` (OPEN),
`E-geometry-create-name-vs-actorname` (OPEN), `E-volume-create-name-vs-volumename`
(OPEN), and the DONE path/param-alias precedents
(`E-blueprint-param-name-path-vs-assetpath`, `E-widget-asset-path-alias-drift`,
`E-material-editor-param-name-drift`) — but this is the **distinct create-verb
split-slot angle** on `material.authoring.create_material` that the DONE material
ticket deliberately left unaddressed, now sharpened by that ticket's own fix.

`material.authoring.create_material` declares the destination as two separate
slots — **`name`** (req) + **`path`** (opt, default `/Game/Materials`) —
`MaterialAuthoringHandler.cpp:361-365`, read at `:374` (`Ctx.RequireString("name")`)
and `:391` (`Ctx.GetString("path", "/Game/Materials")`). There is **no
`assetPath` alias and no single-full-path slot**. So a caller who has one fully
qualified asset path (`/Game/Pickups/Materials/M_StylizedLeaf`) — the exact form
the task brief used ("Create a new material at /Game/Pickups/Materials/M_StylizedLeaf")
— must manually split it into the leaf name and the parent folder before the call
will validate.

The asymmetry is the friction. The DONE `E-material-editor-param-name-drift`
fixed every *operate* verb in this namespace to accept a single **`assetPath`**
(now aliased `assetPath`/`path`/`materialPath` via `MaterialHandlerUtils`, e.g.
`get_material_info` documents "Aliases: `materialPath`, `path`"), and that
ticket **deliberately left `create_material`'s `name`/`path` UNALIASED** —
history #2: *"Left `create_material`/`create_*` folder `path` UNALIASED — it is a
destination folder, not the asset-path slot, so conflating it with
`assetPath`/`materialPath` would be wrong."* That reasoning is sound for the
*folder* `path` slot, but it does not cover the **combined-path guess**: an agent
priming on "every material verb takes one `assetPath`" (now true for the whole
operate surface) reaches for a single `assetPath` on the create verb too, and the
create verb is the one place in the namespace that rejects it — demanding the path
be pre-split into `name`+`path` instead. CLAUDE.md's "camelCase and snake_case
aliases" rule does not cover this: `assetPath` (one combined path) versus
`name`+`path` (split) is a *shape* difference, not a casing variant.

## Repro (verbatim, from the audited M_StylizedLeaf task)

1. `material.authoring.create_material {assetPath:"/Game/Pickups/Materials/M_StylizedLeaf", ...}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'name' (type: string)`
2. One `material.authoring.create_material` **wiki-nav read** to learn the real shape.
3. `material.authoring.create_material {name:"M_StylizedLeaf", path:"/Game/Pickups/Materials", ...}`
   → succeeds. Every later call in the build used the operate-verb `assetPath`
   spelling and succeeded first try.

The error is accurate (the call really is missing `name`) but **misdirecting**
relative to the caller's intent: the caller did not omit a name, they passed the
*whole* path in one arg and expected the verb to split it the way the rest of the
namespace's single-`assetPath` slot reads it. So the agent paid one hard
`MISSING_REQUIRED_PARAM` round-trip plus one wiki-nav read before the working
call — three calls to land a create that the operate-verb mental model says should
be one. Friction note (verbatim): *"first create_material call failed because I
guessed a combined 'assetPath' arg; the method actually wants separate
'name'+'path'. One wiki-nav read fixed it, then everything else was smooth on the
first try."* Zero blocked progress; pure first-call guessability + a discovery
round-trip.

## What it should do

Two non-exclusive options, both preserving the existing `name`+`path` callers:

1. **Accept a combined `assetPath` on the create verbs** and split it
   server-side: if `assetPath` (or `path` ending in an object name) is supplied
   and `name` is absent, derive `name` = leaf, `path` = parent folder
   (`FPackageName::GetLongPackagePath` / `GetShortName`). This makes the create
   verb accept the same single-path shape the operate verbs already accept,
   closing the namespace's last asymmetry. Apply to the `create_*` family that
   shares this slot: `create_material` (`:361`), `create_material_function`
   (`:1336`), `create_material_instance` (`:1533`), and the two domain create
   verbs at `:2231` / `:2270`.
2. If splitting is undesirable, at minimum annotate the `name` slot's
   description (or add an `assetName` alias) and the `path` slot to make the
   "split, do not pass a combined path" contract explicit in the param schema —
   the dispatcher `FParamSpec` alias machinery from
   `E-blueprint-param-name-path-vs-assetpath #4` is the vehicle, same as the
   sibling tickets.

Note this is a closer alias candidate than the deferral in
`E-material-editor-param-name-drift #2` assumed: that ticket reasoned about the
*folder* `path` slot in isolation, before the operate verbs were unified on a
single `assetPath`. Now that the operate surface takes one `assetPath`, the
create verb is the lone outlier, which is exactly the cross-verb-within-namespace
drift the precedent tickets argue should be aliased
(`E-asset-path-vs-assetpath-list-drift` makes the same "list teaches one spelling,
the next verb rejects it" case for `asset.*`).

## Docs angle (`docs/wiki-src/material.authoring.md`)

The overlay's "Limitations" note (line 45) already documents the
`assetPath`/`path`/`materialPath` aliasing across the *sibling operate* RPCs, but
there is **no `### material.authoring.create_material` section** teaching that the
create verb is the exception — that it wants `name`+`path` *split* and rejects a
combined `assetPath`/full-path. The workflow step "Create the material with the
domain-specific create call, or `create_material`…" (line 11) names the verb
without its param shape. Until aliases/splitting land, a short H3 (or a clause on
the existing alias note) stating "the `create_*` verbs take a separate `name` +
optional folder `path`, not a single combined `assetPath` — split the asset path
before calling" closes the discovery gap so the wiki-nav read isn't needed
mid-build. Page to improve: `docs/wiki-src/material.authoring.md`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `material.authoring.set_two_sided` M_StylizedLeaf stylized-leaf task (focus
  `material.authoring.set_two_sided`, outcome `clean`; the seed two-sided
  round-trip and all 14 subsequent calls succeeded — no tool bug, judge filed
  nothing). Distinct PROCESS angle: the very first call,
  `material.authoring.create_material {assetPath:"/Game/Pickups/Materials/M_StylizedLeaf", ...}`,
  hard-failed `[MISSING_REQUIRED_PARAM] Missing required parameter 'name'`,
  cost one wiki-nav read, then succeeded with `{name:"M_StylizedLeaf",
  path:"/Game/Pickups/Materials"}` (3 calls to land 1 create). Source:
  `MaterialAuthoringHandler.cpp:361-365` declares `RPC_PARAM_REQ("name", ...)` +
  `RPC_PARAM_OPT("path", ...)` with no `assetPath` alias; body reads at `:374`
  (`RequireString("name")`) and `:391` (`GetString("path", "/Game/Materials")`).
  The friction is the create verb's split-slot shape vs. the single-`assetPath`
  shape the now-DONE `E-material-editor-param-name-drift` gave every operate verb
  in the same namespace — the create verb is the lone outlier, and that DONE
  ticket (#2) deliberately deferred it. Sibling of the create-verb destination-slot
  tickets `E-level-create-name-path-alias`, `E-geometry-create-name-vs-actorname`,
  `E-volume-create-name-vs-volumename` (all OPEN), but uniquely about the
  *combined-path vs split* guess rather than a casing/synonym alias. Fix:
  accept+split a combined `assetPath` on the `material.authoring.create_*` family
  (`:361`/`:1336`/`:1533`/`:2231`/`:2270`), or alias/document the split contract;
  plus a `docs/wiki-src/material.authoring.md` `### create_material` note.
- `#2-fix` `IN-REVIEW` developer — Implemented the convention this umbrella decides
  for the whole create-* surface (audio/level/volume siblings are blockedBy / mirror
  it): the `material.authoring.create_*` verbs now accept the **combined single-
  `assetPath`** shape every operate verb in the namespace already takes (the repro
  guess), split server-side into leaf name + parent folder, WITHOUT changing the
  meaning of the existing folder `path` slot — so there is no folder-vs-asset
  ambiguity (only an explicit combined `assetPath` is split; folder `path` stays a
  folder) and existing `name`+`path` callers are untouched. This is deliberately the
  light, reusable, per-namespace shape — NOT a fleet-wide rewrite (the geometry
  sibling shipped its own namespace fix the same way; the audio ticket follows once
  this lands). New `Handlers/Material/MaterialCreatePathParamUtils.h` (built on the
  generic `ParamAliasUtils::MakeAliasParamSpec`, mirroring `MaterialHandlerUtils` /
  `GeometryNameParamUtils`): `MaterialCreateNameParamReq` annotates the required
  `name` slot with `assetName`+`assetPath` aliases (so the dispatcher's required-param
  check is satisfied by a combined-`assetPath`-only call and neither alias trips
  UNKNOWN_PARAMS); `MaterialCreateFolderParamOpt` keeps `path` (alias `folder`) as the
  folder slot; `ResolveCreateNameAndFolder(Ctx, DefaultFolder, …)` resolves the leaf +
  folder (explicit `name`/`assetName` wins with folder = `path`/default; otherwise a
  combined `assetPath` is split via `FPackageName::ObjectPathToPackageName` +
  `GetLongPackagePath`/`GetLongPackageAssetName`). Applied to all 8 material create
  verbs in `MaterialAuthoringHandler.cpp` (current HEAD lines — the ticket body's
  `:361`/`:1336`/`:1533`/`:2231`/`:2270` were stale): `create_material` (:465),
  `create_material_function` (:1519), `create_material_instance` (:1753),
  `create_landscape_material` (:2370), `create_decal_material` (:2409),
  `create_post_process_material` (:2448), `create_material_layer` (:3125),
  `create_material_layer_blend` (:3178) — each `RPC_PARAM_REQ("name")` +
  `RPC_PARAM_OPT("path")` spec pair → the helper specs and each
  `RequireString("name")` + `GetString("path", default)` body read →
  `ResolveCreateNameAndFolder`. `add_landscape_layer` (uses `layerName`, not the
  create-asset `name` slot) is correctly left out of scope. Docs: added a
  `### material.authoring.create_material` H3 to `docs/wiki-src/material.authoring.md`
  teaching the split-OR-combined contract (and noting the same applies to the whole
  create-* family), extended the `create_material_instance` H3 Params line, and
  updated the Limitations alias note. Regression test
  `Tests/Material/TestMaterialCreateCombinedAssetPath.cpp`: (1) static — every create
  verb's `name` spec carries the `assetPath`+`assetName` aliases; (2) unit — the split
  helper derives leaf+folder from package-path and object-path forms and reports a
  bare leaf as un-splittable; (3) end-to-end — the real dispatcher accepts
  `create_material {assetPath:"/Game/.../M_Foo"}` (no `name`) without
  MISSING_REQUIRED_PARAM and the asset lands at the split name + folder. Reverting the
  fix fails the static alias check and the dispatch falls back to
  MISSING_REQUIRED_PARAM 'name'. Did not compile/run tests (a later phase does).
