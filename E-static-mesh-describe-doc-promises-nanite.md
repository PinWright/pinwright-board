---
id: E-static-mesh-describe-doc-promises-nanite
title: "static_mesh.describe doc advertises 'Nanite state' it never returns — agents read the doc for NaniteEnabled, find no field, and detour to asset.dump's generic properties.json"
status: OPEN
severity: Low
category: ergonomic
tags: [static-mesh, static-mesh-describe, nanite, asset-dump, dump-parity, registry-tags, docs, misleading-doc]
encounters: 4
lastSeen: 2026-09-03T00:00:00Z
---

# `static_mesh.describe`'s doc promises "Nanite state" that the response (and the `static_mesh.json` sidecar) omit

The `static_mesh` namespace overlay advertises Nanite state verbatim as part of
the shape `static_mesh.describe` returns:

> Dump-parity live read for `UStaticMesh` assets — `static_mesh.describe` returns
> the same shape `asset.dump` writes to `static_mesh.json` (bounds, materials,
> LOD counts, collision, **Nanite state**). …

(`docs/wiki-src/static_mesh.md:3`.)

But the actual `static_mesh.describe` response carries **no Nanite field at
all**. The fields it returns were enumerated verbatim by the parity ticket's own
DONE verify (`E-dump-rpc-parity #4-verify-describe-rpcs`): `bounds`, `materials`,
`lods`, `trianglesByLod`, `verticesByLod` — Nanite is absent. Confirmed in
source: `Handlers/Asset/StaticMeshDescribeHandler.cpp` contains zero `Nanite`
references.

The doc's "same shape as `static_mesh.json`" escape hatch does not save it: the
`static_mesh.json` **dump builder** also emits no Nanite data —
`Handlers/Asset/StaticMeshDumpBuilder.cpp` likewise has zero `Nanite`
references. So both surfaces the doc points at (the live RPC and the sidecar)
omit the very field the doc names. The `E-dump-rpc-parity` matrix compounds the
misdirection: its `static_mesh` row (`#1-initial-audit`, reaffirmed
`#2-priority-correction`) lists the read surface as "bounds, vert/tri count, LOD,
collision, **Nanite status**" — describing a field that was never implemented in
either the sidecar or the describe RPC that ticket shipped.

## Why this is concretely misleading (the recorded friction)

The audited task ("read its current asset info so we know whether Nanite is
already on … note its NaniteEnabled state") reached for the obvious typed
read. Call log shows the agent tried THREE introspection verbs before giving up
on a typed Nanite read:

- `asset.get` → returned only `name/path/class/packagePath`, no tags
  (the established `E-asset-get-doc-promises-tags` friction, recurring here for
  Nanite rather than BlendMode).
- `static_mesh.describe` → friction note verbatim: *"static_mesh.describe (no
  NaniteEnabled field in shape)"* — called precisely because the doc advertised
  "Nanite state," then found none.
- Fell back to `asset.dump`'s generic `properties.json`, reading
  `NaniteSettings.bEnabled` out of the raw UPROPERTY serialization (friction
  note: *"I had to use asset.dump's properties.json (NaniteSettings.bEnabled) as
  the authoritative readback, a discoverability gap"*).

So the doc steered the agent to a verb that does not carry the field, costing an
extra describe round-trip and a sidecar-scraping detour for a value the task's
whole point hinged on. The mutator side already names the field cleanly —
`asset.nanite_rebuild_mesh` returns `naniteEnabled` (`AssetWorkflowHandler.cpp:867`)
and `system.job_status` echoed `naniteEnabled:true` — so the read side is the
only place the obvious "is Nanite on?" question has no typed answer.

## What it should do

Pick one (ergonomic; doc-or-behavior, not a correctness bug):

- **Doc fix (cheap, this ticket's `docs` angle):** strike "Nanite state" from the
  `static_mesh.describe` description in `docs/wiki-src/static_mesh.md` (and drop
  "Nanite status" from the `static_mesh` read surface in `E-dump-rpc-parity`'s
  matrix), and point readers at `asset.dump`'s `properties.json`
  (`NaniteSettings.bEnabled`) as the current authoritative readback. Then the doc
  matches the real `{bounds, materials, lods, trianglesByLod, verticesByLod}`
  output and agents stop calling `static_mesh.describe` expecting `NaniteEnabled`.
- **Behavior fix (closes the gap, makes the doc true):** add a typed `nanite`
  object (`{enabled, keepPercentTriangles, positionPrecision, …}` from
  `UStaticMesh::GetNaniteSettings()`) to **both** the `static_mesh.json` dump
  builder and the `static_mesh.describe` handler via the one shared builder the
  parity ticket mandates — so the field exists in one place and the doc's "same
  shape" claim becomes accurate.

The behavior fix is preferred because Nanite enablement is exactly the kind of
mesh-content baseline `static_mesh.describe` exists to report, and the data is one
`GetNaniteSettings()` call away; the doc fix is the minimal stopgap.

## Distinctness (dedup)

Not a dup of:
- `E-asset-get-doc-promises-tags` (OPEN) — that is `asset.get` advertising
  "registry tags" and the BlendMode material read; this is `static_mesh.describe`
  advertising "Nanite state." Same *pattern* (doc promises a field the verb
  omits), different verb, different field, different overlay page. The two
  together are the recurring "doc over-promises an introspection verb's output"
  theme; cross-referenced, not merged.
- `E-static-mesh-describe-param-name-drift` (OPEN) — that is the `path` vs
  `assetPath` param-alias drift on the same handler; orthogonal to its output
  shape.
- `E-dump-rpc-parity` (DONE) — that ticket *shipped* `static_mesh.describe` but
  never implemented the Nanite field its own matrix promised; this ticket records
  the resulting doc-vs-reality gap. (Reopening that DONE ticket is the wrong
  move; this is the focused doc/field correction.)
- `B-capture-asset-preview-renders-empty` (OPEN, judge-filed for this task) — the
  separate blank-render capture bug.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a DemoRoom Nanite
  enable-and-verify task (seed `render.nanite_rebuild_mesh`, which worked;
  outcome clean for the rebuild). Distinct PROCESS angle from the judge's
  `B-capture-asset-preview-renders-empty`: the readback detour. The agent needed
  `NaniteEnabled` per the story, called `static_mesh.describe` because
  `docs/wiki-src/static_mesh.md:3` advertises "Nanite state," found none
  (friction verbatim: "static_mesh.describe (no NaniteEnabled field in shape)"),
  and fell back to `asset.dump` `properties.json` `NaniteSettings.bEnabled`.
  Source-confirmed: `StaticMeshDescribeHandler.cpp` and `StaticMeshDumpBuilder.cpp`
  both contain zero `Nanite` references, so neither the live RPC nor the
  `static_mesh.json` sidecar the doc points at carries the advertised field; the
  `E-dump-rpc-parity` matrix likewise overstates "Nanite status" for the
  static_mesh read surface. Fix: either strike "Nanite state" from the overlay
  (doc) or add a typed `nanite` object to the shared static-mesh builder
  (behavior). Cross-ref `E-asset-get-doc-promises-tags` (same doc-over-promise
  pattern on `asset.get`).
- `#2-additional-repro-post-rebuild` `OPEN` reporter — Additional evidence
  (catalog-thumbnails task, realism mode): reproduced the exact gap right after a
  confirmed Nanite enable. `render.nanite_rebuild_mesh` on
  `/Engine/BasicShapes/Cube.Cube` completed via `system.job_status` with
  `result:{naniteEnabled:true, rebuilt:true}`, yet `static_mesh.describe` on the
  same Cube returned, verbatim, no nanite key:
  `{"bounds":…,"materials":[…],"lods":1,"trianglesByLod":[48],"verticesByLod":[54],"lightmapResolution":64,"collision":{…},"collisionTraceFlag":"CTF_UseDefault"}`.
  Then ran `asset.dump` on the Cube and read the freshly-written
  `Saved/PinWright/asset-dumps/Engine/BasicShapes/Cube/static_mesh.json` — it too
  omits any nanite field (and `static_mesh.txt` likewise), confirming the doc's
  "same shape as `static_mesh.json`" escape hatch still does not hold even
  immediately post-rebuild. So an agent asked to "enable Nanite then verify it
  still looks correct" has no typed Nanite readback on the surface the
  `static_mesh` overlay names. Same verb / field / overlay page as
  `#1-initial-audit`; no new file.
- `#3-level-scope-coverage-has-no-verb-either` `OPEN` reporter — Third encounter,
  and it widens the gap from one asset to a whole level. An ENV blind-A/B critic
  pass over `/Game/FPS/Maps/FPS_Compound` had to score the stream against a
  "Nanite meshes everywhere geometry allows" quality bar, i.e. it needed **Nanite
  coverage across 1023 actors / 18 kit meshes**, not one asset's flag. Three
  surfaces were tried and none answers it:
  `static_mesh.describe {assetPath:"/Game/FPS/Env/Meshes/SM_ENV_WallPanel"}`
  returned verbatim
  `{"bounds":…,"materials":[…],"lods":1,"trianglesByLod":[160],"verticesByLod":[314],"lightmapResolution":4,"collision":{…},"collisionTraceFlag":"CTF_UseSimpleAndComplex","rebuildRenderConsumers":{…}}`
  — no nanite key, reproducing `#1` and `#2` on UE 5.8;
  `geometry.audit_static_meshes {folder:"/Game/FPS/Env/Meshes"}` ran ten checks
  (`inverted`, `inconsistent_winding`, `not_closed`, `degenerate_triangles`,
  `non_manifold`, `empty`, `mirrored_build_scale`, `thin_shell`, `z_fighting`,
  `floating_components`) and **none of them is about Nanite, LOD policy or
  rendering cost**; `level.audit` over the 1023-actor world likewise offers no
  Nanite/LOD/draw-cost check in its check list. So the per-asset doc-vs-reality gap
  this ticket already records has a level-scope twin: there is no verb that answers
  "what fraction of this level's placed geometry is Nanite?", which is a standard
  acceptance question for any UE5 environment review. Falling back to
  `asset.dump` `properties.json` per asset does not scale to a kit, and reading
  `NaniteSettings` out of the `.uasset` bytes with `grep -a` was inconclusive here
  (the property name did not appear as a plain string in any of the 18 files).
  Adds weight to this ticket's preferred **behavior fix** (a typed `nanite` object
  on the shared static-mesh builder) and asks that whichever surface gains it also
  be reachable in bulk — a `nanite` column on `geometry.audit_static_meshes` rows,
  or a Nanite check in `level.audit` — so coverage is one call rather than N. No
  new file: same verb, same field, same overlay page as `#1`/`#2`.
- `#4-weapons-critic-fourth-encounter` `OPEN` WEAPONS-critic — Fourth encounter, on a weapons
  asset this time, reproducing `#1`–`#3` verbatim on UE 5.8.
  `static_mesh.describe {assetPath:"/Game/FPS/Weapons/Meshes/SM_WPN_AR"}` returned exactly these
  keys and no others: `bounds`, `materials`, `lods`, `trianglesByLod`, `verticesByLod`,
  `lightmapResolution`, `collision`, `collisionTraceFlag`, `rebuildRenderConsumers` — **no
  `nanite` field**. The wiki page the reviewer landed on is the generated
  `Saved/PinWright/wiki/static_mesh.describe.md`, whose summary still says the verb returns
  "bounds, materials, LOD counts, lightmap resolution, and collision, **plus Nanite state**", so
  the over-promise reaches the reader through the *generated method page*, not only through the
  `docs/wiki-src/static_mesh.md:3` overlay this ticket already cites — worth noting for whoever
  takes the doc-fix branch, since striking the phrase upstream is what regenerates that page.
  No new file: same verb, same field, same claim, already covered by `#1`–`#3`. This encounter
  adds no new mechanism, only reach — the field is now recorded as missing across a Game weapons
  mesh, a Game env mesh, a Game DemoRoom asset and `/Engine/BasicShapes/Cube`, i.e. every asset
  class anyone has pointed the verb at.
