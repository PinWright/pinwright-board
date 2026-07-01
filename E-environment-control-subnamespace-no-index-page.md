---
id: E-environment-control-subnamespace-no-index-page
title: "environment.md advertises call(\"environment.control\"), which is not a registered node and 404s to a did-you-mean list — the overlay prelude promises a handle that does not resolve"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, wiki, discoverability, environment, overlay, did-you-mean]
---

# The `environment` overlay prelude advertises `call("environment.control")`, but that path is not a registered node and resolves to a did-you-mean fallback

The `environment` namespace overlay (`docs/wiki-src/environment.md`, line 5;
this prelude also seeds the generated `wiki-generated/environment.md`) tells
callers, verbatim:

> Use `call("environment.build")` for generated environment assets or snapshots,
> and `call("environment.control")` for live editor-world controls such as
> console commands, skylight, sun intensity, and time-of-day changes.

But the two halves of that sentence resolve **asymmetrically**, and the cause is
*registration*, not the presence/absence of an "index page":

- `call("environment.build")` resolves cleanly because `environment.build` IS a
  registered method — `REGISTER_RPC_HANDLER("environment.build", "environment", ...)`
  at `Handlers/Environment/EnvironmentHandler.cpp:524` (a legacy-dispatcher
  leaf). `WikiHandler::ClassifyNode` checks `MethodsByLowerName` first
  (`Catalog/WikiHandler.cpp:210`) and returns `Method`, so `RenderMethodPage`
  emits a normal **method page** (not a namespace/sub-namespace index).
- `call("environment.control")` does **not** resolve because there is no
  `environment.control` node of any kind. The wiki tree is built from each
  handler's **Category** field, not from method-name prefixes
  (`WikiHandler.cpp:138-159` walks the dotted *Category* string). All four
  control methods register Category `"environment"`
  (`set_time_of_day` :692, `set_sun_intensity` :765, `set_skylight_intensity`
  :828, `console_command` :895), so the only node produced is `environment` —
  there is no `environment.control` Category node, Topic node, or registered
  method. `ClassifyNode("environment.control")` therefore finds it is not a
  Method, not a Topic, `bIsCategory=false`, `bHasDeeperCategory=false` (no
  Category starts with `environment.control.`) and returns `NotFound`
  (`WikiHandler.cpp:239`), so `RenderPage` falls to the did-you-mean branch
  (`WikiHandler.cpp:537-538`).

So `environment.control` is not a real "sub-namespace" in this tree — it is only
a naming convention *inside* method names; the Category-based tree has no node
for it and no `environment.control.md` page is generated (only the four
per-method pages `environment.control.console_command.md`,
`environment.control.set_time_of_day.md`, `environment.control.set_sun_intensity.md`,
`environment.control.set_skylight_intensity.md`). An agent that follows the
overlay's own discovery instruction takes a dead-end detour: it lands on
`environment.md`, reads "call `environment.control`", does so, gets a did-you-mean
list rather than the four control methods, and recovers via the four full dotted
names that the auto-generated `## Methods` index already listed on the same
`environment.md` page it came from. The methods all dispatch fine by full name,
so this is process/discoverability cost only — one extra wiki-nav round-trip,
zero failed RPCs — hence Low severity. The fix is purely a docs/overlay defect,
not a router or generator defect.

This is distinct from the three nearby tickets:
- **`B-wiki-namespace-underscore-not-found`** — that's the `NormalizeQuery`
  first-underscore→dot mangling of single-token *underscore* namespaces
  (`game_framework`, `world_partition`). `environment.control` is a multi-segment
  *dotted* token with no underscore; it is not mangled, it simply has no index
  page because there is no registered leaf by that exact name.
- **`E-wiki-page-filenames-dotted-flat-undiscoverable`** — that's the direct
  filesystem-Read path (nested-vs-flat filename guessing). This is the live
  `call()` router path following an explicitly advertised doc-fetch handle.
- **`B-export-snapshot-empty-stub`** — that's the export/import stub tool bug
  (already filed; this task's golden-hour evidence is appended there at `#4`).
  This ticket is purely about the `environment.control` doc-fetch detour.

## Fix

**Docs-only (recommended, root cause).** Stop advertising a handle that does not
resolve. Edit the `environment` overlay prelude
(`docs/wiki-src/environment.md`, line 5) to drop `call("environment.control")`
and instead point callers at the full `environment.control.<method>` names —
which already appear in the auto-generated `## Methods` index on the very page
the agent came from, and dispatch fine by full name. This is a one-line prose
edit, no code, no risk, and is the proportionate response for a Low / no-failure
discoverability detour. `call("environment.build")` may keep being advertised (it
is a real registered method).

Why **not** the heavier code options: making `call("environment.control")`
resolve to a synthetic "sub-namespace index" — by (a) emitting an
`environment.control.md` index page, or (b) teaching the router to list
`<prefix>.*` methods for any dotted method-name prefix — invents a
sub-namespace-index concept the Category-based tree does not have (the tree is
built from handler **Category**, and every control method's Category is plain
`environment`). Both would touch the load-bearing discovery core
(`WikiHandler.cpp` / `WikiDiskGenerator.cpp`) for a cosmetic, zero-RPC-failure
detour, and (b) would have to iterate `MethodsByLowerName` keys for the
`environment.control.` prefix since `environment.control` is neither a Category
node nor in `AllNodes`. Disproportionate churn on discovery code for tiny payoff;
keep them out of scope.

Surface to edit: `docs/wiki-src/environment.md` (the overlay that seeds both the
source and generated namespace page — line 5's "call `environment.control`"
sentence).

## Evidence

Struggle-audit of a clean golden-hour/dusk sky-rig task (namespace
`environment`, outcome otherwise driven by the `export_snapshot` bug; the spawns,
tuning, and controls all succeeded first try). The discovery flow recorded four
wiki-nav calls before the first execute: `(root index)` → `actor` →
`environment.control` (**"wiki-nav (got did-you-mean list)"** per the call log)
→ `environment.build`. The `environment.control` nav is the only one that
returned a did-you-mean fallback; the immediately-following `environment.build`
nav rendered cleanly (a method page — `environment.build` is a registered
method) — the exact asymmetry above. Friction note (verbatim):
"Spawns/tuning/controls were smooth on the first try (wiki was accurate ...)";
the lone navigation detour was the `environment.control` did-you-mean. Confirmed
against the committed tree: `wiki-generated/environment.build.md` exists (a
generated method page) but there is **no** `wiki-generated/environment.control.md`
— and crucially no `environment.control` node of any kind, because the four
control methods all register Category `environment`, not `environment.control`.
Only the four `environment.control.*` per-method pages exist, and
`docs/wiki-src/environment.md` line 5 advertises `call("environment.control")`
regardless. One extra wiki-nav round-trip, no failed RPC — process cost only.

## History
- `#1-initial-audit` `OPEN` reporter — Process/docs friction from a clean golden-hour sky-rig task (namespace `environment`). The `environment` namespace page (`docs/wiki-src/environment.md` line 5 / generated `wiki-generated/environment.md` line 7) advertises `call("environment.control")` for live-world controls, but no `environment.control` leaf is registered and no `environment.control.md` index page is generated, so the live `call()` router returns a did-you-mean list instead of the four `environment.control.*` methods. Asymmetric with its sibling `call("environment.build")`, which resolves cleanly because `environment.build` is a registered legacy-dispatcher leaf with a generated `environment.build.md`. Call log shows the agent's discovery flow drilled `environment.control` and got "did-you-mean (list)" (recorded verbatim), then recovered via the already-rendered `## Methods` list / full dotted names — all controls dispatched first try, process cost only (Low). Distinct from `B-wiki-namespace-underscore-not-found` (underscore-token mangling), `E-wiki-page-filenames-dotted-flat-undiscoverable` (direct-Read filename guessing), and `B-export-snapshot-empty-stub` (the export stub bug, separately filed). Proposed fix: generate an `environment.control.md` sub-namespace index (mirror `environment.build.md`), or have the router list `<prefix>.*` methods for a registered dotted prefix, or drop the misleading "call `environment.control`" instruction from the `environment.md` overlay.
- `#2-reword-and-docs-fix` `IN-REVIEW` developer — REWORD then docs-only fix. Validity: defect is real (the `environment` overlay prelude advertises `call("environment.control")`, which is not a registered node and 404s to a did-you-mean list), but the original ticket framed it around a fictional "sub-namespace index page" — this is a Category-based tree (`WikiHandler.cpp:138-159` walks the handler **Category** string, not method-name prefixes), and all four control methods register Category `environment` (EnvironmentHandler.cpp:692/765/828/895), so no `environment.control` node of any kind exists; `environment.build` resolves to a normal **method page** (registered method, EnvironmentHandler.cpp:524, ClassifyNode→Method at WikiHandler.cpp:210/RenderMethodPage), not an "index page." Rewrote title/body/Fix/tags to match the Category-based model and demoted the two code options (synthesize an index page / list `<prefix>.*` methods) as disproportionate churn on the discovery core for a Low/no-failure detour. Applied the recommended root-cause fix: `docs/wiki-src/environment.md` line 5 no longer advertises `call("environment.control")`; it now points callers at the four full `environment.control.<method>` names (which already appear in the auto-generated `## Methods` index on that same page and dispatch fine). Files: `docs/wiki-src/environment.md` (overlay prelude). No production C++ changed and no router/generator behavior changed, so no automation regression test applies — the wiki tree regenerates from this overlay at editor launch; verification is reading the regenerated `wiki-generated/environment.md` prelude.
