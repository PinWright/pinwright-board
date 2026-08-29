---
id: B-pcg-method-page-contradicts-namespace-overlay
title: "The served `pcg.create_graph` page states the wrong error contract — CLASS_NOT_FOUND for the disabled-plugin case, which the namespace page explicitly denies — and it is the page the error response links to; `pcg.add_node`'s page documents no error codes at all, from the same missing-H3 cause"
status: OPEN
severity: Medium
category: bug
tags: [pcg, create_graph, add_node, wiki, docs, error-contract, plugin-disabled, class-not-found, overlay, divergence, served-page]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Two served pages, one namespace, opposite error contracts

`Saved/PinWright/wiki/pcg.create_graph.md`, the whole of what it says about failure — it is the
`graphClass` param row and there is no `## Notes` block on the page:

> `graphClass` (`string`, optional): UPCGGraph subclass to instantiate: full path
> (/Script/<Module>.<Class>) or short class name. Resolved by reflection, so any loaded subclass
> works without PinWright linking its module. **Rejected with CLASS_NOT_FOUND when it does not
> resolve**, is not a UPCGGraph subclass, or is abstract. The resolved class is echoed back as
> graphClass. Default: `/Script/PCG.PCGGraph`.

`Saved/PinWright/wiki/pcg.md`, `## Conventions`, third bullet:

> The stock UE 5.8 subclass is `/Script/ProceduralVegetation.ProceduralVegetationGraph`, which needs
> the experimental Procedural Vegetation Editor plugin enabled — with it disabled the class does not
> resolve and the call returns **`PLUGIN_DISABLED` naming the plugin, not `CLASS_NOT_FOUND`**.

The namespace page is right. `PCGGraphCreate.cpp:52-57` calls
`PinWrightPCG::SendPluginDisabledForScriptPath` on the resolve failure and returns before the
`CLASS_NOT_FOUND` branch at `:61-68`. The per-method page describes only that later branch and
presents it as the whole contract — *"Rejected with CLASS_NOT_FOUND when it does not resolve"* is
exactly the case `pcg.md` singles out as **not** CLASS_NOT_FOUND.

**Why this one matters more than a thin page: the per-method page is what the error response links
to.** A caller who receives a refusal from `pcg.create_graph` follows that link and is told, at the
moment they are debugging, that an unresolvable class is a class-resolution problem. The correct
answer — enable a plugin — is on a page they have no reason to open.

`Saved/PinWright/wiki/pcg.add_node.md` documents **no error codes at all**. It is the H1, the
namespace line, the summary and four param rows; there is no `## Notes` block, no error section, and
none of its four param descriptions names a code. `pcg.add_node` emits `PLUGIN_DISABLED` (the
`NamesDisabledPlugin` test pins it), `CLASS_NOT_FOUND`, and more.

## One cause for both halves

`RenderMethodPage` (`Source/PinWright/Private/Catalog/WikiHandler.cpp:741-757`, re-derived at HEAD)
builds a method page from the registry — H1, namespace, escaped summary, `RenderParamList` — and
then appends exactly one thing, `:749-755`:

```cpp
const FString MethodSection = WikiOverlay::LoadMethodSection(Reg.MethodName);   // :749
if (!MethodSection.IsEmpty())
{
    Out += TEXT("\n## Notes\n\n");
    Out += MethodSection;
```

`WikiOverlay::LoadMethodSection` (`Catalog/WikiOverlay.cpp:206`) reads the `### <ns>.<method>` H3
section out of the namespace overlay file.

`Docs/wiki-src/pcg.md` contains exactly **one** `###` heading in the whole file — `### pcg.generate`
at `:57`. So every other `pcg.*` method page is registry text and nothing else, and its error
contract is whatever the handler's `RPC_PARAM` strings happen to mention. For `create_graph` that is
a partial contract asserted as complete; for `add_node` it is nothing. Same absence, same file, same
fix location — which is why both halves are in one ticket rather than two.

**Note where the divergence puts the plugin's own honesty tests:** `pcg.md:12`'s claim is correct
for the full `/Script/` spelling and wrong for the short one, because the guard early-returns on
anything without that prefix — `E-plugin-disabled-guard-misses-short-class-name`, filed separately.
So the per-method page is accidentally right about the short-name case and wrong about the case
`pcg.md` documents, and the namespace page is the reverse. Neither page is correct on its own; a
fixer writing the `### pcg.create_graph` section must state both branches.

## Distinct from — and this needs saying, or a triager will merge them

`E-build-query-overlay-not-merged-into-served-page` (IN-REVIEW, Low) is the canonical statement of
the mechanism above: its `#2` establishes that `RenderMethodPage` serves per-method overlay content
**only** from `### <ns>.<method>` H3 sections inside the namespace file. That analysis is what this
ticket cites, and it is correct. (Its citation `Catalog/WikiHandler.cpp:410-426` is stale against
HEAD — the function now sits at `:741-757`. Recorded here rather than edited into that file, which
is another agent's.)

But that ticket **explicitly rejected a systemic diagnosis** and concluded:

> The defect is one mis-placed overlay file, not a routing failure.

Its case is *omission*: rich content existed, was authored into the wrong file shape, and reached no
page — the served page was thin, and nothing on it was untrue. **This case is the one it ruled out.**
The content exists, is correct, and is served at namespace level; the per-method page says something
**different** about the same failure. Divergence, not omission. The two need separate fixes: theirs
was to relocate a standalone file into an H3 section; this one needs a new `### pcg.create_graph`
section written, because no correct per-method text exists anywhere to relocate — and no amount of
relocating fixes a page that carries a contradicting claim of its own from the handler registry.

Also distinct:

- `B-error-code-adoption-test-scans-comments` (IN-REVIEW, Medium) — the error-code coverage guards
  are **source** scans over handler `.cpp` text (`Tests/Core/TestErrorCodeRegistry.cpp:6-11`:
  registration and adoption). They compare emitted codes against `ErrorCodes.h`; nothing checks that
  a *served wiki page* documents the codes its method emits. So no existing guard could fail on
  `pcg.add_node.md` having no error codes, and none could notice `pcg.create_graph.md` naming the
  wrong one. Cited as the reason this class survives, not as an overlapping defect.
- `E-wiki-page-filenames-dotted-flat-undiscoverable` (IN-REVIEW, Low) — a *filename* discoverability
  ticket, which asserts of the page content: *"The content is correct and complete once found."*
  This finding contradicts that sentence directly, and the quote is included so a fixer of either
  ticket sees it. Different surface (where the file is versus what it says), so not merged; but the
  claim should not survive unqualified once this is known.
- `B-wiki-cache-frozen-after-first-render` — a staleness/caching defect. Both pages here are freshly
  generated and consistent with their sources; the sources disagree with each other.

## What it should do

Add a `### pcg.create_graph` H3 section to `Docs/wiki-src/pcg.md` giving the full error contract —
`PLUGIN_DISABLED` for a class that fails to resolve because its plugin is off, `CLASS_NOT_FOUND` for
a class that is genuinely absent, not a `UPCGGraph` subclass, or abstract, and `INVALID_PATH` /
package-name refusals — and correct the `graphClass` param string at `PCGGraphCreate.cpp:32-33`,
which is the half that is served when no overlay section exists. Add a `### pcg.add_node` section
covering its codes. Both land on the served pages through the existing `LoadMethodSection` →
`RenderMethodPage` path with no generator change, the same way
`E-build-query-overlay-not-merged-into-served-page`'s fix did.

Worth considering in the same pass, but **not** claimed by this ticket: a contract test that a
method page names the codes its handler emits would turn this whole class from "noticed by a caller
mid-debug" into a suite failure. That is a larger design question than one namespace's docs.

## Dedup

Board-wide search across all statuses for `pcg.create_graph` / `pcg.add_node` documentation tickets
and for wiki page-content tickets: the wiki family is
`E-build-query-overlay-not-merged-into-served-page`, `E-wiki-page-filenames-dotted-flat-undiscoverable`,
`B-wiki-namespace-underscore-not-found`, `B-wiki-cache-frozen-after-first-render`,
`E-wiki-root-index-omits-standalone-guides`, `E-wiki-maturity-tiers` and
`E-wiki-model-authoring-page-too-large` — each a different surface, and none about a served page
contradicting its own namespace overlay. Nothing owns this.

## History
- `#1-served-pages-disagree` `OPEN` reporter — Verified against the generated tree and the sources. `Saved/PinWright/wiki/pcg.create_graph.md` states its entire error contract in the `graphClass` param row — "Rejected with CLASS_NOT_FOUND when it does not resolve, is not a UPCGGraph subclass, or is abstract" — and carries no `## Notes` block. `Saved/PinWright/wiki/pcg.md` `## Conventions` (source `Docs/wiki-src/pcg.md:12`) says the opposite for the disabled-plugin case: "the call returns `PLUGIN_DISABLED` naming the plugin, not `CLASS_NOT_FOUND`". The namespace page matches the code — `PCGGraphCreate.cpp:52-57` routes through `SendPluginDisabledForScriptPath` and returns before the `CLASS_NOT_FOUND` branch at `:61-68`. The per-method page is the one the error response links to, so the wrong contract is served exactly when a caller is debugging. `Saved/PinWright/wiki/pcg.add_node.md` documents no error codes at all: H1, namespace, summary, four param rows, no `## Notes`, no code named in any param description — while the verb emits `PLUGIN_DISABLED` (pinned by `PinWright.pcg.add_node.NamesDisabledPlugin`) and `CLASS_NOT_FOUND`. Shared root cause, which is why both halves are one ticket: `RenderMethodPage` (`Catalog/WikiHandler.cpp:741-757`, re-derived at HEAD) appends per-method overlay content only from `WikiOverlay::LoadMethodSection` (`Catalog/WikiOverlay.cpp:206`) at `:749-755`, which reads `### <ns>.<method>` H3 sections out of the namespace file — and `Docs/wiki-src/pcg.md` has exactly one `###` in the whole file, `### pcg.generate` at `:57`. Every other `pcg.*` page is therefore registry text alone: for `create_graph` a partial contract asserted as complete, for `add_node` none. Explicitly distinguished from `E-build-query-overlay-not-merged-into-served-page` (IN-REVIEW, Low), whose mechanism analysis this cites and whose conclusion — "The defect is one mis-placed overlay file, not a routing failure" — rules out its own case: that one is content *absent* from a page (thin, nothing untrue), this one is content that exists correctly at namespace level and is *contradicted* by the per-method page. Divergence, not omission; theirs was fixed by relocating a file, this needs a section written, because no correct per-method text exists to relocate and the contradicting claim comes from the handler registry itself. Noted while re-deriving: that ticket's `WikiHandler.cpp:410-426` citation is stale (now `:741-757`); recorded here, not edited into another agent's file. Also cross-linked `B-error-code-adoption-test-scans-comments` — the error-code guards (`Tests/Core/TestErrorCodeRegistry.cpp:6-11`) are source scans over handler `.cpp` text and check registration/adoption only, so nothing verifies that a served page documents the codes its method emits, which is why this class survives — and `E-wiki-page-filenames-dotted-flat-undiscoverable` (IN-REVIEW, Low), whose "The content is correct and complete once found" this contradicts, quoted so a fixer of either sees it. Severity Medium: impact class is Medium — a soft blocker, since a caller following the served contract reaches the wrong diagnosis and recovers only by reading a second page or the source — not High, because no RPC returns wrong data and the failing call does refuse honestly. Reach modifier declined in both directions and the choice differs deliberately from the sibling `E-plugin-disabled-guard-misses-short-class-name` (Low): that one needs the experimental off-by-default Procedural Vegetation plugin to reproduce and is a genuine rare edge path, whereas `pcg.create_graph` and `pcg.add_node` are the two entry verbs of the namespace and every pcg session hits them — so no bump down, and no bump up either, because `pcg.*` is gated on an optional engine plugin and does not run in almost every session globally.
