---
id: E-wiki-maturity-tiers
title: "Wiki index and namespace pages carry no maturity tier (core/experimental/internal)"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [wiki, docs, discoverability, maturity]
---

# Wiki index and namespace pages carry no maturity tier (core/experimental/internal)

The pinwright.com docs and the Fab listing classify every namespace by maturity
(website `src/_data/namespaces.js` `status` field: 27 `core`, 37 `exp`, 2 `int`
across 66 entries, rendered as Core/Experimental/Internal badges + filter chips).
The AI-facing wiki served by `call()` exposes none of this: the root namespace
index and per-namespace pages have no stage marking at all. An agent planning
work cannot tell a solid surface (`blueprint`, `asset`) from a
still-changing one (`material`, `niagara`, the whole AI/gameplay group) without
stumbling into scattered prose caveats (`asset.md` Chooser "IN-REVIEW",
`gas.md`/`gameplay_tags.md` "IN-REVIEW").

Root cause: the wiki pipeline has no slot for maturity anywhere.

- Namespaces are not registered — they exist only as dotted prefixes of handler
  `Category` strings (`WikiHandler.cpp:151-165` prefix walk). No namespace
  metadata struct exists.
- `FHandlerRegistration` (`Handlers/HandlerRegistration.h:13-20`) has no
  tag/flag/maturity field; `REGISTER_RPC_HANDLER` cannot carry one.
- Index descriptions come solely from overlay preludes
  (`WikiOverlay::LoadGroupPrelude` → `RenderRootNamespaceEntry`,
  `WikiHandler.cpp:72-101`); overlays in `docs/wiki-src/` have no frontmatter
  support (`WikiOverlay::ParseInto`, `WikiOverlay.cpp:70-156`).
- The plugin README maturity table (`README.md:11-21`) is hand-written prose by
  loose "Area", not namespace slug, and is never read by the generator.

The website classification was authored independently during launch prep and
never ported back, so only marketing surfaces have it.

**Fix:** add a single per-namespace maturity map as a data file (e.g.
`docs/wiki-src/maturity.json`, slug → `core|experimental|internal`), seeded from
the website's `namespaces.js` `status` values (the canonical, complete source —
the Fab listing is a lossy prose summary). Load it alongside the wiki cache and
render at the two existing sites:

- `RenderRootNamespaceEntry` (`WikiHandler.cpp:72-101`): suffix the index entry,
  e.g. `` - `material` (experimental) — ... ``. Plain text, not badges — the
  consumer is an agent.
- `RenderNamespaceHeader` (`WikiHandler.cpp:423-433`): a `Stability:` line with
  a one-line meaning ("works but still changing" etc.).

Log a warning for a registered namespace absent from the map and for map
entries matching no namespace (drift detection; plugin namespace list and the
website's 66 slugs are not guaranteed 1:1 — reconcile at seed time).

Alternative considered: overlay frontmatter per `docs/wiki-src/<ns>.md`.
Co-locates stage with prose but requires a new frontmatter branch in
`ParseInto`, touches ~60+ files, and leaves namespaces without overlay files
(the bare-index-entry case, e.g. `game_features`) with nowhere to declare a
tier. The single data file avoids both.

## History
- `#1-initial-report` `OPEN` reporter — Root index and namespace pages expose no core/experimental/internal tier; classification exists only on pinwright.com (`namespaces.js` status field) and in the README prose table, neither consumed by the wiki generator. Proposed data-file map + two render-site changes.
- `#2-maturity-map-implemented` `IN-REVIEW` developer — "Seeded `docs/wiki-src/maturity.json` (65 entries, exactly the top-level namespaces derived from handler Category strings: 27 core / 37 experimental / 1 internal). `Catalog/WikiHandler.cpp`: new `LoadMaturityMap()` + `MaturityBySlug` on `FWikiCache`; `RenderRootNamespaceEntry` gained a Tier param appending `(experimental)`/`(internal)` after the backticked name (core/unmapped stay bare); `RenderRoot` gained the one-line marker legend; `RenderNamespaceHeader` emits a `Stability:` line keyed on the first dotted segment so nested pages inherit the top-level tier. Drift detection at cache build UE_LOGs a Warning per registered namespace missing from the map and per map key matching no namespace, gated to the live dispatcher so test-fixture registries don't spam. Seed reconciliation: plugin-only `game_features` defaulted to experimental; website slugs `log` and `debug` match no plugin namespace and were omitted. Tests: three new cases in `Tests/Infra/TestWikiHandler.cpp` (root markers+legend, Stability lines incl. nested inheritance, unmapped-namespace-renders-unmarked via fixture dispatcher). Not compiled per project rule; next test-loop run validates."
