---
id: B-maturity-unmapped-namespace-fails-open
title: "Wiki maturity map fails open: an unmapped namespace renders as core"
status: IN-REVIEW
severity: High
category: bug
tags: [wiki, maturity, docs, fail-open, discoverability, registry-json]
---

# Wiki maturity map fails open: an unmapped namespace renders as core

`docs/wiki-src/maturity.json` classifies each top-level namespace `core` |
`experimental` | `internal`, and the wiki renders that tier on the two surfaces
an agent actually reads. The map **failed open**: a namespace with no entry
rendered exactly like a `core` one, so a brand-new namespace — or one someone
forgot to classify — silently advertised itself as the most-trusted tier. The
default landed on "most trusted" instead of "unknown", which is the wrong
direction for a trust signal.

There is no fallback assignment anywhere that says "core". The equation is
assembled out of three separate places, which is why it was invisible:

1. **Load** — `WikiHandler.cpp:136` `LoadMaturityMap()` returns a `TMap` with no
   entry for an unmapped slug (and an empty map on a missing/unparseable file).
2. **Root-index marker** — `WikiHandler.cpp:643-645` `RenderRoot` passed
   `Tier ? *Tier : FString()`, and `WikiHandler.cpp:84`
   `RenderRootNamespaceEntry` collapsed the two cases together:
   `const FString Marker = (Tier.IsEmpty() || Tier == TEXT("core")) ? FString() : ...`.
   Unmapped and `core` produce a byte-identical entry.
3. **The legend that assigns the meaning** — `WikiHandler.cpp:640` prints
   *"unmarked namespaces are core"*. That sentence is what turns "no marker"
   into a positive claim of the most-trusted tier. Without it the omission
   would merely be silent; with it, the page states something false.

Two more consumers inherit the same default:

- **Namespace page** — `WikiHandler.cpp:748-763` `RenderNamespaceHeader` wraps the
  whole `Stability:` block in `if (const FString* Tier = ...Find(TopLevel))` with
  no `else`, so an unmapped namespace page carries **no** stability line at all
  and reads as core against the root legend. A tier string outside the known set
  (a typo in the JSON) took the same silent path.
- **`registry.json`** — `WikiHandler.cpp:925-928` `GetRegistryStats` left
  `Stat.Tier` empty (`WikiHandler.h:29`: *"empty when maturity.json has no
  entry"*), publishing a blank tier into the file `CLAUDE.md` calls the single
  generated source of truth for the website and store listing. A downstream
  consumer is free to read blank as core.

Drift detection existed but is not a gate: `WikiHandler.cpp:242-247` emits a
`LogWikiHandler` **Warning** per unmapped namespace, only when the cache is
built from the live dispatcher, at editor startup, into a log with thousands of
lines. Nothing fails.

**The defect was pinned by a test.** `TestWikiHandler.cpp:1416`
`PinWright.infra.wiki_handler.Maturity.UnmappedNamespaceUnmarked` asserted the
fail-open behaviour deliberately — `TestFalse("unmapped probe namespace carries
no tier marker")` and `TestFalse("unmapped namespace page carries no Stability
line")` — so the suite locked it in and a fix would have read as a regression.
It came in with `E-wiki-maturity-tiers`, whose own `#2` history entry records
the choice as *"core/unmapped stay bare"*.

**Namespaces actually unmapped at the time of filing: exactly one — `_test`.**
Derived two ways that agree: `rg -U -o 'REGISTER_RPC_HANDLER\(\s*"[^"]*"\s*,\s*"([^"]*)"'`
over `Source/` yields 69 top-level categories against 68 `maturity.json` keys,
and the generated `Saved/PinWright/wiki/registry.json` (68 namespaces, every
`tier` non-empty) matches the map exactly because `GetRegistryStats` filters
`_test` out. `_test` is *not* filtered from the root index, so
`Saved/PinWright/wiki/index.md:31` shipped `` - `_test` — Internal smoke-test
endpoints … not a stable API `` with no marker — a namespace whose own prose
says it is not stable, rendered in the tier that means solid primary surface.
The realized blast radius was therefore one dev-only namespace; the severity
below is argued from the failure direction, not from that count.

**Severity: High.** By the rubric's impact classes this is *"silent wrong data on
a normal path (the caller trusts a result that is a lie and builds on it)"* — the
tier is a trust signal, the root index is the first page every agent reads, and
the wrong value is the one that says "trust this most". The reach modifier
argues up (the root index runs in essentially every session), but `Critical` is
defined as editor crash or a write that corrupts asset data and this writes
nothing, so it stops at High rather than being bumped into a class it does not
belong to. The counter-argument for `Medium` is that only `_test` was actually
mis-advertised and it self-describes in prose; that argues about today's count,
which the rubric treats as an `encounters`-style tiebreak, not as impact.

**Fix:** make the map fail closed and gate the omission at authoring time.

- One resolver, `ResolveTier` (`WikiHandler.cpp:200`), is now the only read of
  the map for rendering. A slug with no entry — or an entry whose value is not
  `core`/`experimental`/`internal` — resolves to `unclassified`, never to empty
  and never to `core`. All three render sites go through it, so they cannot
  disagree about what an omission means.
- Only `core` renders bare (`WikiHandler.cpp:103`). `unclassified` is marked
  inline like any other non-core tier, and the root legend
  (`WikiHandler.cpp:680`) explains it.
- `RenderNamespaceHeader` (`WikiHandler.cpp:789-806`) now emits a
  `Stability: unclassified` line instead of nothing.
- `registry.json` publishes `"tier": "unclassified"` rather than a blank
  (`WikiHandler.cpp:972`).
- `unclassified` is a render-time fallback only; it is never a legal value to
  write into `maturity.json`.
- `_test` classified `internal` — the one unmapped namespace, and `internal`
  ("plumbing, not intended for direct use") is what its own prelude describes.

**Not done, and why:** the drift check was left at `Warning` rather than promoted
to `Error`. UE's automation framework fails any test that emits an uncaptured
`Error`, and the wiki cache rebuilds on a registry-generation bump — which can
land inside an unrelated test and fail it for something it did not cause. The
hard gate is a test instead:
`PinWright.infra.wiki_handler.Maturity.EveryRegisteredNamespaceIsClassified`
(`TestWikiHandler.cpp:1485`) renders the live root index and fails naming every
namespace that came back `(unclassified)`. Refusing outright in
`WikiDiskGenerator` was also rejected: it would take the whole discovery surface
down over one missing docs entry.

## History
- `#1-fail-open-default` `OPEN` reporter — Maturity map fails open. Unmapped namespace renders byte-identically to `core` on the root index (`WikiHandler.cpp:84` collapses `Tier.IsEmpty()` into the `core` branch; `:640` legend then asserts "unmarked namespaces are core"), renders no `Stability:` line at all on its namespace page (`:748-763`, `if (Find(...))` with no `else`), and publishes a blank `tier` into `registry.json` (`:925-928`, `WikiHandler.h:29`). Drift is a live-dispatcher-gated `UE_LOG` Warning (`:242-247`), not a gate. `Maturity.UnmappedNamespaceUnmarked` (`TestWikiHandler.cpp:1416`) asserted the behaviour deliberately, so the suite locked the defect in. Exactly one namespace was unmapped — `_test` — and it shipped unmarked at `Saved/PinWright/wiki/index.md:31` despite its own prelude saying "not a stable API".
- `#2-fail-closed-and-gated` `IN-REVIEW` developer — "Added `ResolveTier` (`WikiHandler.cpp:200`) as the single map read for rendering: missing entry or unknown value resolves to `unclassified`, never empty and never `core`. `RenderRootNamespaceEntry` (`:103`) now renders bare only for `core`; root legend (`:680`) explains `(unclassified)`; `RenderNamespaceHeader` (`:789-806`) emits `Stability: unclassified — … less stable than experimental` instead of nothing; `GetRegistryStats` (`:972`) publishes `unclassified` instead of a blank. Drift warning now also catches an unknown tier value and names the consequence. Classified the one unmapped namespace: `_test` -> `internal` in `docs/wiki-src/maturity.json` (68 -> 69 keys). Tests: `Maturity.UnmappedNamespaceUnmarked` renamed to `Maturity.UnmappedNamespaceRendersUnclassified` and its two `TestFalse` assertions flipped to `TestTrue` on the marker and the Stability line, plus an assertion that the line explains the tier; new `Maturity.EveryRegisteredNamespaceIsClassified` fails the suite naming any namespace that renders `(unclassified)`; `Maturity.RootIndexMarkers` gained a legend assertion. Net +1 automation test. Docs in the same commit: plugin `CLAUDE.md` (Wiki Authoring Constraints), `Docs/rpc-design.md` (checklist item + §13), `Docs/wiki-src/README.md` (authoring rules), `Docs/wiki-src/wiki.md` (agent-facing marker legend), `WikiHandler.h:29` comment. Compile-checked both touched .cpp with `-SingleFile`: `[1/1] Compile [x64] WikiHandler.cpp` / `TestWikiHandler.cpp`, `Result: Succeeded` each. Full suite not run — a peer agent owns the build/suite this wave."
