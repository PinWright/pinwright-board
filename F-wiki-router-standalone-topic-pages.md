---
id: F-wiki-router-standalone-topic-pages
title: "Wiki router cannot resolve standalone topic pages (e.g. runtime-uobject-inspection)"
status: DONE
severity: Medium
category: feature
tags: [wiki, router, catalog, topic-pages, discoverability]
---

# Wiki router cannot resolve standalone topic pages (e.g. runtime-uobject-inspection)

The wiki router (`Catalog/WikiHandler.cpp::GetWikiCache`) builds its addressable
surface exclusively from the live RPC registry, so standalone topic pages such
as `docs/wiki/runtime-uobject-inspection.md` cannot be reached via
`call("<page>")` — only cross-references from registered namespace pages route
to them. Add a topic-page registry so any `docs/wiki/<slug>.md` whose filename
does not match a registered Category gets indexed as a `EWikiNodeKind::Topic`
node, returned by `WikiOverlay::LoadGroupPrelude(slug)` directly, and included
in the fuzzy-suggestion pool. Acceptance: `call("runtime-uobject-inspection")`
returns the page contents; `call("runtime-uo")` lists it as a fuzzy
suggestion; the topic-page registry is auto-discovered from the wiki directory
at subsystem init, no manual list maintained.

## History
- `#1-routing-gap` `OPEN` reporter — The wiki router's cache builder in `Catalog/WikiHandler.cpp::GetWikiCache` iterates only the live RPC registry (Category and Method names), so any `docs/wiki/<slug>.md` whose filename doesn't match a registered Category is invisible to `call("<slug>")` — the call returns `Not found` with fuzzy suggestions drawn from the same registry-derived pool. `F-wiki-runtime-uobject-inspection` was resolved with a docs-only workaround (advertise `runtime-uobject-inspection.md` from every plausible namespace overlay so cross-refs reach it) pending this fix, because true routing requires source edits to the catalog/wiki-handler that were out of scope for that documentation-only ticket.
- `#2-topic-page-registry` `IN-REVIEW` developer — Added `EWikiNodeKind::Topic`; `GetWikiCache` now enumerates `<plugin>/docs/wiki/*.md`, skips `README*` and Category collisions, adds standalone slugs to `TopicNodes` and `SuggestionPool`; `ClassifyNode` returns `Topic` for matching slugs; new `RenderTopicPage` emits `# <slug>` + `LoadGroupPrelude(slug)`; `wiki.get` routes the `Topic` arm. Regression test `TestWikiHandler.cpp` covers resolve / fuzzy / category-not-shadowed.
- `#3-verify-topic-routing` `DONE` tester — Verified: `call("runtime-uobject-inspection")` rendered `# runtime-uobject-inspection`, and `call("runtime-uo")` returned `runtime-uobject-inspection` in the fuzzy suggestion list.
