---
id: E-wiki-page-filenames-dotted-flat-undiscoverable
title: "Wiki per-method pages use dotted-flat filenames (ns.method.md); direct nested-path Reads 404"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, wiki, discoverability, filename-layout, generated-wiki]
encounters: 2
lastSeen: 2026-06-23T10:08:23Z
---

# Wiki per-method pages use dotted-flat filenames (ns.method.md); direct nested-path Reads 404

When an agent reads the generated wiki pages directly off disk (rather than
through `call("<page>")`), the on-disk layout is **flat with dotted filenames**
— `niagara.create_system.md`, `niagara.authoring.md`, etc. — all siblings in one
directory, not nested under a `niagara/` folder. The intuitive guess is the
nested form (`niagara/create_system.md`), so the first direct Reads return
Not-Found and the agent has to fall back to a `Glob` to locate the real file
before it can read any per-method doc.

This is purely a discoverability cost on the **direct filesystem read** path
(distinct from `B-wiki-namespace-underscore-not-found`, which is about the live
`call()` router mangling underscore namespaces — a different surface). The
content is correct and complete once found; the only friction is that the
flat-dotted naming convention is undocumented, so the first guess at a page path
misses.

## What it should do

Either (a) document the flat-dotted filename convention where the wiki layout is
described (so a reader knows pages live at `<dir>/<namespace>.<method>.md`, all
siblings, no nesting), or (b) have the wiki entry-point / index page list the
exact on-disk page paths it cross-references, so an agent reading directly never
has to guess-then-Glob. A README or header note in the generated wiki root
stating "pages are flat: `<namespace>.<method>.md`" would remove the miss.

Docs surface to improve: the generated wiki root index / README (the page an
agent lands on first), and optionally a one-line layout note in
`docs/wiki-src/wiki.md` (the mcp-usage guide overlay).

## Evidence

Struggle-audit of a clean `niagara.create_system` authoring task (10/10 calls
ok, outcome clean; one `niagara` wiki-nav up front). Friction note, verbatim:
"wiki files use dotted flat filenames (`niagara.create_system.md`) so my first
nested-path Reads 404'd and I needed one Glob to find them." Confirmed against
the committed overlay tree: `docs/wiki-src/` pages are themselves flat-dotted
(`niagara.authoring.md`, `niagara.compile-state.md`, `niagara.dump-files.md`,
`niagara.graph.md`, `niagara.nir.md`), matching the generated layout the agent
hit. One extra Glob round-trip; no failed RPC — process cost only, hence Low
severity. Likely recurs for any namespace whose docs are read directly off disk,
so worth a single layout note rather than per-namespace fixes.

## History
- `#1-initial-audit` `OPEN` reporter — Process/docs friction from a clean `niagara.create_system` task. Generated wiki per-method pages use flat dotted filenames (`niagara.create_system.md`) as directory siblings, but the agent's first direct Reads assumed nested paths (`niagara/create_system.md`) and 404'd, forcing a `Glob` to locate the files. Distinct from `B-wiki-namespace-underscore-not-found` (that's the live `call()` router; this is the direct filesystem-Read path). Verified the committed `docs/wiki-src/` tree is itself flat-dotted, matching the generated layout. Proposed fix: document the flat-dotted page-path convention at the wiki root index / README (and optionally `docs/wiki-src/wiki.md`) so direct readers don't guess-then-Glob. Docs-only; not namespace-specific.
- `#2-recurs-geometry-extrude` `OPEN` reporter — Same friction recurs on a different namespace, confirming it is namespace-independent (as predicted in #1). Struggle audit of a clean `geometry.extrude` pedestal-blockout task (9/9 calls ok, outcome clean apart from the judge-filed `B-extrude-inset-empty-selection-whole-mesh`). Friction note, verbatim: *"wiki page paths are dotted single files (geometry.create_box.md) not nested folders, so my first Read attempts (geometry/create_box.md) 404'd and I needed a Glob to find the real layout."* Identical guess-then-Glob round-trip as #1, now on `geometry.*` rather than `niagara.*` — second independent occurrence reinforces fixing this once at the wiki root index / README rather than per-namespace. Still Low (one extra Glob, no failed RPC).
- `#3-hint-states-flat-layout` `IN-REVIEW` developer - Implemented (uncommitted) as part of the `E-call-unknown-arg-fields-silently-ignored` change set: `GWikiDocHint` (McpRequestCore.cpp) now states the layout on every wiki-discovery reference response - "every wiki page is on disk in the 'wiki' directory, flat (no subfolders): root index 'index.md', namespace pages '<namespace>.md', method pages '<namespace.method>.md' (dotted filenames verbatim)". This lands the note on the exact response an agent reads before its first direct page Read, covering proposal (a). The generated root index page content itself was not changed; if the verifier judges a static note in the root index/README is also needed, return this ticket. Not yet compiled - verification pending alongside the parent change set.
- `#4-suite-green-committed` `IN-REVIEW` developer - Parent change set compiled clean and full `PinWright` suite green on UE 5.6 (3428 passed, 0 failed); pushed as `77856a0a` on plugin master. The hint text (incl. the flat-naming sentence) is asserted non-empty by the wiki-reference tests; live check of the rendered hint left for mcp-review.
