---
id: E-wiki-model-authoring-page-too-large
title: "`model.authoring` is a 65 KB single-file wiki page — reading it whole stalled an agent for 600 s and cost a restart; it is the first page every mesh-authoring agent opens"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [wiki, docs, model-authoring, pwmodel, page-size, wiki-disk-generator, discoverability, read-budget]
encounters: 1
lastSeen: 2026-08-27T18:58:19+05:00
---

# The primary `.pwmodel` guide is too large to read in one call

`Saved/PinWright/wiki/model.authoring.md` is **67,009 bytes** (65.4 KiB) on this
checkout. An agent that read it whole stalled for **600 s** with no progress and had to
be restarted — a full agent restart, not a retry.

That page is the primary authoring guide for `.pwmodel`, so it is the *first* page a
mesh-authoring agent opens, which means this is the first thing such an agent hits. The
MCP server's own instructions make the on-disk wiki the default discovery workflow
("Read `Saved/PinWright/wiki/index.md`, then use filesystem search and file-reading tools
in `Saved/PinWright/wiki/` as the default discovery workflow"), so the whole-file read is
the behaviour the plugin asks for.

## Measured on this checkout, 2026-08-27

| page | bytes |
|---|---|
| **`model.authoring.md`** | **67,009** |
| `geometry.md` | 36,142 |
| `render.md` | 32,800 |
| `material.authoring.md` | 30,041 |
| `index.md` | 29,574 |
| `skeleton.md` | 26,592 |
| `landscape.md` | 25,758 |
| `unattended.md` | 25,089 |

`model.authoring.md` is the largest wiki page by **1.85x** over the runner-up, and is
larger than the next two pages combined. It is an outlier, not the norm — which is what
makes splitting it a bounded change rather than a wiki-wide reformat.

**A line cap does not bound it.** The page is only **357 lines**; its longest line is
**1,551 characters**. So a reader that limits by line count reads the whole thing anyway.
The cost is bytes, and nothing on the read path caps bytes.

## The generator already does exactly the split this needs — for a different page kind

The wiki is generated at editor launch by
`Plugins/PinWright/Source/PinWright/Private/Catalog/WikiDiskGenerator.cpp` from
`Plugins/PinWright/docs/wiki-src/`. Its own header banner says so:

```cpp
// WikiDiskGenerator.cpp:25-26
return TEXT("<!-- GENERATED from docs/wiki-src overlays + handler registry at editor launch. ")
    TEXT("Do not edit here; edit the matching source under docs/wiki-src/. -->\n\n") + Body;
```

Output directory, `WikiDiskGenerator.cpp:164`:

```cpp
Dir = FPaths::ProjectSavedDir() / TEXT("PinWright") / TEXT("wiki");
```

And the machinery for section-level splitting is already there. `WikiDiskGenerator.cpp:20`:

```cpp
// docs/wiki-src/<slug>.md, but a method page's editorial lives as an H3 section in
// its namespace file, ...
```

A **namespace** page's H3 sections are already lifted into their own per-method pages.
`model.authoring` is a **topic** page, so nothing splits it: it has **15 `##` sections and
zero `###` sections**, which is why it comes out as one 65 KB file. The section boundaries
are clean and already authored —

    Authoring contract / The minimal file / Validate, compile, inspect / Document structure /
    Transform spaces / Skinning: the space, the aiming, and what stays rigid / Faceted shading /
    Seeded variation / Scalloped and lobed shapes / Materials / Collision /
    Primitive and response readings that are not what the name suggests /
    Re-compiling and provenance / What `pwmodel 0` does not have / See also

## What it should do

Emit a **section index plus per-section pages** for oversized topic pages, the way the
generator already emits per-method pages from a namespace file's H3 sections — e.g.
`model.authoring.md` becomes a short index listing the 15 sections, with the bodies at
`model.authoring.collision.md`, `model.authoring.skinning.md` and so on. The landing page
then costs a few KB and the agent reads only the section it needs.

Cheapest useful variant if the split is unwanted: have the generator refuse to emit a page
over some byte budget without a section index, or have the page open with a
"sections and their byte sizes" table so a reader can pick a slice up front. Either removes
the whole-file read; the split removes it permanently.

## Workaround

Read it in slices (`sed -n`, `Read` with `offset`/`limit`) or `grep` for the section
needed. This works, and it is what the agent did after the restart — but it requires
already knowing the page is oversized, which is exactly what a first-time reader does not
know.

## Precedent and neighbours — cited as precedent, not as duplicates

- `E-asset-dump-oversized-fields` (**DONE**, Medium) is the same *shape* — "too large for
  one Read" — and was accepted at Medium. It is scoped to asset-dump sidecars under
  `.editor-automation/asset-dumps/` and is closed. This ticket is the same failure on the
  **generated wiki**, a surface that ticket never touched.
- `E-wiki-page-filenames-dotted-flat-undiscoverable` (IN-REVIEW, Low) is the same file
  family, but its friction is *finding* the page (nested-path guesses 404 against the
  flat-dotted layout). Once found, its page reads fine. Here the page is found
  immediately and reading it is what fails. Orthogonal; fixing either leaves the other.
- `E-wiki-root-index-omits-standalone-guides` (DONE) and `F-wiki-router-standalone-topic-pages`
  (DONE) are routing/index reachability for standalone topic pages, not page size.
  `E-wiki-maturity-tiers` (IN-REVIEW) is per-page maturity labelling. None of the 20+
  existing wiki tickets mentions page byte size, page splitting, or a per-section index.

severity rationale: impact=friction, not correctness — the content is right and a sliced read gets it (Low) x reach=bumped one level: this is the primary `.pwmodel` guide and the largest page on the wiki, so every mesh-authoring agent opens it first and hits this first, and the observed cost was not a re-read but a 600 s stall and a full agent restart -> Medium

## History
- `#1-65kb-page-stalled-an-agent` `OPEN` reporter — Found while building the Atlantis example level on host project EAContentExamples58 (map as forcing function; see that project's `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. An agent reading `Saved/PinWright/wiki/model.authoring.md` in one call stalled 600 s with no progress and had to be restarted. Measured on disk: **67,009 bytes**, 357 lines, longest line 1,551 chars — so a line-count cap does not bound the read; only bytes do. It is the largest generated wiki page by 1.85x (next: `geometry.md` 36,142, `render.md` 32,800, `material.authoring.md` 30,041, `index.md` 29,574), i.e. an outlier rather than a wiki-wide condition. Source page is `Plugins/PinWright/docs/wiki-src/model.authoring.md` (66,860 bytes); the generated copy is written by `Catalog/WikiDiskGenerator.cpp` (banner at `:25-26`, output dir `FPaths::ProjectSavedDir()/"PinWright"/"wiki"` at `:164`). The split mechanism already exists for a different page kind — `WikiDiskGenerator.cpp:20` records that a method page's editorial is lifted from an **H3 section** of its namespace source — but `model.authoring` is a topic page with 15 `##` sections and zero `###` sections, so nothing splits it. Proposed: emit a section index plus per-section pages for oversized topic pages, reusing that machinery; the 15 H2 boundaries are already authored. Worked around by slicing the read; defect untouched.
- `#2-split-oversized-pages-at-h2` `IN-REVIEW` developer — "Extended `Catalog/WikiDiskGenerator.cpp` so a rendered page over the new `kSectionSplitBudget` (40 KB, declared in `WikiDiskGenerator.h`) is no longer written as one file: `WikiDiskGenerator_SplitOversizedPages` keeps the page's opening prose, appends a `## Sections` index, and emits each `## ` section as `<slug>.<section-slug>.md` beside it — the `## ` analogue of the `### ` method-section lift the generator already did. Splitting happens on raw render bodies before the generated-file header is prepended, so section pages carry the same header; fence-aware so a `## ` inside a ``` block is not a boundary; the root index is never split and a namespace page's auto-generated `## Subgroups` / `## Methods` indexes stay on its landing page. `model.authoring` (68,209 chars, 15 `## ` sections, zero `###`) becomes a ~2 KB landing page plus 15 section pages; it is the only page on this tree over the budget (next largest 36,142), so the change is bounded to the outlier. `call(\"model.authoring.collision\")` resolves because `McpRequestCore` maps a wiki path to the on-disk file by existence, not by renderer classification. Added `PinWright.infra.wiki_disk_generator.OversizedPageSplitsIntoSectionIndex` in `Tests/Infra/TestWikiDiskGenerator.cpp`: no page but the root index may exceed the budget while it still has 2+ `## ` boundaries, every `## Sections` entry must name a file that exists on disk with a body, and at least one index must be emitted. Added a stripped-at-render `<!-- -->` authoring note to `docs/wiki-src/model.authoring.md` recording that `## ` headings are now page boundaries and that a `### ` here would swallow everything below it. NOT COMPILED and NOT RUN — build/test loop pending."
