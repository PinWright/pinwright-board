---
id: E-niagara-add-renderer-class-no-example
title: "niagara.add_renderer rendererClassPath has no wiki example or enum of valid renderer classes"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, niagara, add_renderer, discoverability, class-path]
encounters: 2
lastSeen: 2026-06-24T09:27:12Z
---

# niagara.add_renderer rendererClassPath has no wiki example or enum of valid renderer classes

`niagara.add_renderer` requires a `rendererClassPath` argument, but neither the
RPC schema string nor the `niagara` wiki overlay tells the caller what to put
there. The schema (`NiagaraEditHandler.cpp:1810`) describes the param only as
`"Renderer properties class path"`, and `docs/wiki-src/niagara.md` documents
`set_property` and `set_parameter` with concrete JSON examples but gives **no
`add_renderer` example** — so there is no listed value, no enumeration of the
valid renderer classes (Sprite / Mesh / Ribbon / Light / Component / Decal
properties), and no note that the resolver accepts a **short class name**, not
just a fully-qualified `/Script/...` path.

The resolver is permissive — `ResolveRendererClass` →
`NiagaraEdit::ResolveNiagaraSubclass<UNiagaraRendererProperties>(path, "Niagara")`
(`NiagaraEditHandler.cpp:1484-1486`) accepts both `"NiagaraSpriteRendererProperties"`
and `"/Script/Niagara.NiagaraSpriteRendererProperties"`. But because nothing
documents this, the caller has to go discover a valid class name out-of-band
before the very first `add_renderer` call can be written. On a wrong/empty value
the call returns `RENDERER_CLASS_NOT_FOUND` / `INVALID_ARGUMENT` with no
suggestion of valid classes.

## What it should do

Add an `add_renderer` example to the `niagara` overlay page (and/or a one-line
"valid renderer classes" list) so the class value is discoverable from the wiki
alone — e.g. document that `rendererClassPath` accepts a short name and list the
common `UNiagaraRendererProperties` subclasses (`NiagaraSpriteRendererProperties`,
`NiagaraMeshRendererProperties`, `NiagaraRibbonRendererProperties`,
`NiagaraLightRendererProperties`, `NiagaraComponentRendererProperties`,
`NiagaraDecalRendererProperties`). Optionally, on `RENDERER_CLASS_NOT_FOUND`,
echo the resolvable subclass names in the error so the surface is
self-correcting.

Docs page to improve: `docs/wiki-src/niagara.md` (the `### Individual edit
examples` block — add an `add_renderer` JSON example alongside the existing
`set_property` / `set_parameter` ones).

## Evidence

Struggle-audit of a `niagara.create_system` end-to-end authoring task
(create system + standalone emitter, add handle, **add sprite renderer**,
compile, save, inspect, validate, spawn preview actor — 10 calls, all
`ok:true`, outcome clean). Friction note, verbatim: "the sprite renderer
rendererClassPath had no wiki example so I confirmed
`/Script/Niagara.NiagaraSpriteRendererProperties` via a read-only engine-header
Glob." The whole task otherwise ran clean on the first try; this is the one
spot where the agent had to leave the documented surface (engine-header Glob) to
write a required argument value. Process cost only — one extra discovery step,
no failed call — hence Low severity.

## History
- `#1-initial-audit` `OPEN` reporter — Process/docs friction surfaced by a clean `niagara.create_system` authoring task (10/10 calls ok, outcome clean). `niagara.add_renderer` requires `rendererClassPath` but `docs/wiki-src/niagara.md` ships no `add_renderer` example and the schema string is just "Renderer properties class path", so the agent had to Glob engine headers to confirm `/Script/Niagara.NiagaraSpriteRendererProperties` before writing the call. Resolver (`NiagaraEditHandler.cpp:1484`) actually accepts short names too, but that's undocumented. Proposed fix: add an `add_renderer` example + list of valid `UNiagaraRendererProperties` subclasses to the niagara overlay page; optionally echo resolvable class names on `RENDERER_CLASS_NOT_FOUND`. Docs-only; named overlay `docs/wiki-src/niagara.md`.
- `#2-second-occurrence-move-renderer-task` `OPEN` reporter — Recurrence on a different clean task (focus `niagara.move_renderer`: build `/Game/VFX/NS_Campfire`, add Sprite/Light/Ribbon renderers, reorder, compile, save — 15 calls, all ok, outcome clean). The agent hit the exact same gap on the **three** `add_renderer` calls: friction note verbatim — "the wiki didn't give an example rendererClassPath, so I confirmed the /Script/Niagara.* class names from the engine source." Confirms this is not a one-off (now 2 tasks): every multi-renderer authoring flow pays the same out-of-band class-name discovery cost on the first `add_renderer`, and here for all three renderer kinds (`NiagaraSpriteRendererProperties`, `NiagaraLightRendererProperties`, `NiagaraRibbonRendererProperties`) at once. Strengthens the case to list the common subclasses in the overlay example. Still Low (process only, no failed call), still docs-only on `docs/wiki-src/niagara.md`.
