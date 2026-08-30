---
id: E-actor-name-resolution-label-collision
title: "actorName slot is documented 'Display label or name' but never tells callers the unique internal object name is the safe disambiguator when labels collide"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [actor, actorname, add_tag, remove_tag, set_transform, find_by_tag, display-label, disambiguation, discoverability, docs]
---

# `actorName` resolution against colliding display labels is undocumented — the unique object name as the safe key is left for the caller to discover

Every per-actor `actor.*` verb (`add_tag`, `remove_tag`, `set_transform`,
`get`, `get_transform`, `duplicate`, …) takes a singular `actorName` slot whose
parameter help is uniformly **"Display label or name of … actor"** (e.g.
`Handlers/Actor/LifecycleHandler.cpp:29` for `actor.delete`). That phrasing presents "display
label" and "name" as interchangeable, and says nothing about what happens when
multiple actors in the world **share the same display label** — which is common
for placed copies (all seven PointLights in the test level carry the identical
label `TestPointLight`).

When labels collide, passing the shared label is ambiguous, and nothing in the
docs tells the caller how to resolve it deterministically. The working answer —
discovered empirically in this task — is to use each actor's **unique internal
object name** (`PointLight_1` … `PointLight_7`, i.e. the *leaf* of the object
path that `actor.find_by_class` / `actor.find_by_tag` already return in their
`path` field). That internal name is the safe disambiguator, but the "Display
label or name" wording neither names it nor flags it as the collision-safe
choice, so the caller has to infer that "name" specifically means the object-path
leaf and that it is guaranteed unique.

This is the **input-side / discovery** half of the colliding-label problem and is
distinct from `E-actor-verification-actorpath-is-map-path` (already filed by the
per-finding judge), which is the **output-side** bug: that the verification
block's `actorPath` field returns the map package path instead of the actor
object path. Even after that output bug is fixed, a caller still has to *know*
to disambiguate via the object name on the way in — that knowledge gap is what
this ticket tracks. It is also distinct from `E-actor-select-singular-actorname-rejected`
(singular-vs-array arity) and `E-inspect-find-by-tag-internal-name-not-label`
(the inspect-twin `name`-field internal-vs-label asymmetry); neither documents
how to *resolve* an ambiguous `actorName` when labels collide.

## What it should do

Document on the `actor.*` overlay page (`docs/wiki-src/actor.md`) — which today
has per-method sections only for a handful of verbs (describe / get_components /
spawn_from_blueprint / add_component), **none** for `find_by_tag` / `add_tag` /
`remove_tag` / `set_transform` / `get`, and **never** explains `actorName`
resolution or that display labels are non-unique — that:

1. `actorName` accepts either the display label **or** the unique internal object
   name (the object-path leaf, e.g. `PointLight_1`);
2. display labels are **not** guaranteed unique (placed copies routinely share
   one); and
3. when labels collide, pass the **unique internal object name** — exactly the
   leaf of the `path` returned by `actor.find_by_class` / `actor.find_by_tag` —
   as the collision-safe disambiguator.

A one-line note in the shared `actorName` parameter help ("Display label OR the
unique internal object name; use the object name when labels collide") would also
close the gap at the point of use.

## Evidence (this task — lighting-cleanup tag-grouping, seed `actor.find_by_tag`)

Friction note, verbatim: *"Minor: all 7 PointLights share the display label
\"TestPointLight\", so label-based actorName for add_tag/remove_tag/set_transform
was ambiguous; I disambiguated via each actor's unique internal name
(PointLight_1..7, the object-path leaf), which resolved correctly — but the
wiki's \"display label or name\" wording doesn't explicitly flag the unique
object name as the safe disambiguator when labels collide."*

The task's 18-call log shows the workaround inline: after `actor.find_by_class`
(className=`/Script/Engine.PointLight`) returned seven same-labelled lights, all
seven `actor.add_tag` calls and the later `remove_tag` / `set_transform` /
`get` / `get_transform` calls were keyed on the object-name leaves
`PointLight_1`…`PointLight_7` rather than the shared label — a successful run, but
the caller had to derive the disambiguation strategy itself because the docs
don't state it.

**Workaround:** When display labels are non-unique, identify actors by the unique
object-path leaf (returned in the `path` field of `actor.find_by_class` /
`actor.find_by_tag`) and pass that as `actorName`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a lighting-cleanup tag-grouping task (seed `actor.find_by_tag`; 18 calls, all `ok`, no retries — clean OUTCOME, PROCESS friction only). All seven PointLights shared the display label `TestPointLight`, making label-based `actorName` ambiguous; the auditor disambiguated via the unique internal object name (`PointLight_1..7`, the object-path leaf from `find_by_class`/`find_by_tag`) and it resolved correctly, but the `actorName` help ("Display label or name") never names the object name as the collision-safe key, and `docs/wiki-src/actor.md` has no section explaining `actorName` resolution at all. Distinct PROCESS angle from the judge-filed `E-actor-verification-actorpath-is-map-path` (that is the OUTPUT `actorPath`-value bug; this is the INPUT-side discoverability gap — how to resolve an ambiguous `actorName`). Dedup (ripgrep over OPEN+closed; qmd unavailable): not covered by `E-actor-select-singular-actorname-rejected` (arity), `E-inspect-find-by-tag-internal-name-not-label` (inspect-twin `name`-field asymmetry), or `E-actor-verbs-reject-actorpath-slot` (input-key drift). Proposed: document on `docs/wiki-src/actor.md` (and in the shared `actorName` param help) that display labels are not unique and the unique internal object name (object-path leaf) is the collision-safe disambiguator.
- `#2-doc-fix` `IN-REVIEW` developer — Implemented the docs/ergonomic fix. (1) Added a new `## actorName resolution & colliding labels` section to `docs/wiki-src/actor.md` (a top-level `##` section, so it renders onto the `actor` namespace page via `FWikiOverlay`): documents that `actorName` resolves against display label / internal object name (`GetName()`) / object path; that display labels are NOT guaranteed unique (placed copies share one); and that the collision-safe key is the unique internal object name — the object-name leaf returned in the `path` field of `actor.find_by_class`/`actor.find_by_tag`. This is the consolidated canonical input-side statement (also covers the sibling docs gaps without four separate carve-offs). (2) Appended a collision-safe note to the shared `actorName` parameter help at the point of use across the per-actor verbs — `ActorPropertyHandler.cpp` (add_tag/remove_tag/get_components/etc.), `ActorTransformHandler.cpp` (set_transform/get_transform/get_bounding_box), `ComponentHandler.cpp` (set_component_properties/get_component_property), `QueryHandler.cpp` (`actor.get`): "Display labels are not unique; when labels collide, pass the unique internal object name (the object-path leaf, e.g. PointLight_1) as the collision-safe key." Verified against source: `McpActorUtils::FindActorByName` (Utils/ActorUtils.cpp:69-74) matches label OR `GetName()` OR path and `break`s on the FIRST label match, so a colliding label resolves non-deterministically while the unique `GetName()` resolves exactly — the technical premise the docs now state. Note: the recommended flow keys off the `path` field of `find_by_class`/`find_by_tag`, which already returns the real actor object path today and is NOT changed by the output-side `E-actor-verification-actorpath-is-map-path` fix (which only touches the verification block's `actorPath` field) — so this doc does not depend on that ticket and is not timing-coupled. Regression test: `FWikiHandlerActorNameLabelCollisionDocumentedTest` in `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` renders the `actor` namespace page and the `actor.add_tag` method page through production `WikiHandler::RenderPage` and asserts the overlay-exclusive markers ("actorName resolution", "not guaranteed unique", "internal object name", "object-name leaf" on the namespace page; "collision-safe key" in the method page's auto Parameters list) — fails if either the actor.md section or the param-help note is reverted. Also fixed in passing the stale `LifecycleHandler.cpp:24-27` citation (real: `Handlers/Actor/LifecycleHandler.cpp:29`) and the overstated "no section at all" wording.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
