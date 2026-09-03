---
id: E-wiki-root-index-omits-standalone-guides
title: "Wiki root index lists no standalone guide pages, so task-shaped guides are unfindable"
status: DONE
severity: Medium
category: ergonomics
tags: [wiki, catalog, discoverability, topic-pages, root-index]
---

# Wiki root index lists no standalone guide pages, so task-shaped guides are unfindable

`WikiHandler::RenderPage("")` assembled the root index from only the prelude of
each top-level namespace overlay. Standalone topic pages (`workflows`,
`level-building`, `level-review`, `visual-review`, `asset-audit`,
`safe-mutation-save`, `runtime-uobject-inspection`, `unattended`, `wiki`, ...)
auto-enrol into `TopicNodes` and are navigable via `call("<slug>")`, but nothing
listed them on the front page: an agent found one only if some namespace prelude
happened to link it, which is manual and silently rots. These guides are the
highest-value pages for a newcomer and were the only ones invisible.

Acceptance: the root index carries a compact one-line-per-page index of
standalone guide pages, derived from the live topic enrolment (no hand-maintained
list in code), excluding namespace and method pages, with `workflows` first.

## History
- `#1-root-index-gap` `OPEN` reporter — Root index (`RenderRoot` in `Catalog/WikiHandler.cpp`) enumerates only `ImmediateChildren` of the namespace tree; `Cache.TopicNodes` never reaches it. `call()` therefore advertises 66 namespaces and zero guides.
- `#2-task-guides-section` `IN-REVIEW` developer — Added `RenderGuideIndex` / `GuideSlugs` / `GuideSummary` to `Catalog/WikiHandler.cpp` and emitted a `## Task guides` section from `RenderRoot`, above `## Namespaces` (the tier legend moved down to sit with the namespace list it annotates). Guide slugs = `Cache.TopicNodes` entries with no dot (a dotted topic is namespace-owned reference material already linked from its parent page); `workflows` sorts first, rest alphabetical. Each entry is one line: slug + a summary derived from the page's own prelude (first sentence, cut to its headline clause when over 140 chars, hard-truncated at 200) - never the page's prelude, so the root's byte budget is respected (~1.5 KB for 14 guides). Tests: `PinWright.infra.wiki_handler.RootIndex.ListsGuidePages` (section present, workflows first, no dotted/namespace entries, per-entry length cap, and each summary verified to occur verbatim on the page it describes - the pin against a hardcoded list) and `PinWright.infra.wiki_disk_generator.RootGuideListPagesExistOnDisk` (every listed guide is materialized as `<slug>.md` beside `index.md`). Shared `ExtractSection` / `ParseSlugBullets` helpers added to `Tests/Infra/WikiDocTestHelpers.h`. Not yet built or run - needs an editor start plus the automation suite.
- `#3-runtime-verified` `DONE` tester — Built (UE 5.8 `Result: Succeeded`, zero errors AND zero
  warnings in both the console log and UBT's `Log.txt`, `-DisableAdaptiveUnity` confirmed by 0
  standalone `.cpp` compile actions, fresh DLL timestamp — not the exit code) and runtime-verified
  by starting the editor so the wiki regenerated. `Saved/PinWright/wiki/index.md` now carries
  `## Task guides` above `## Namespaces`, 14 entries, `workflows` first then alphabetical, 33,130 B
  total (was 31,628). Every listed slug has a `<slug>.md` beside it. Both new tests pass
  (`infra.wiki_handler.RootIndex.ListsGuidePages`, `infra.wiki_disk_generator.RootGuideListPagesExistOnDisk`)
  along with the pre-existing `wiki_handler.Maturity.*` and the generator byte-equality/idempotency
  tests, so the new section did not disturb them. **Known cosmetic interaction, deliberately not
  filtered:** the two rename redirect stubs (`level-blockout`, `blockout-review`) are themselves
  dot-free topic pages, so they list alongside their replacements — four entries where two would do.
  Each is self-labelling ("**Renamed — this guide is now X**"), and both stubs are scheduled for
  deletion once the rename settles, at which point the index self-corrects with no code change.
  Suppressing them would need either sniffing the summary prose (couples the index to one page's
  wording, defeating the derived-not-listed design) or a new wiki-authoring marker; neither is worth
  it for a temporary page. Committed as `1f840652`.
