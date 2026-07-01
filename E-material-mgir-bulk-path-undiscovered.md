---
id: E-material-mgir-bulk-path-undiscovered
title: "The material.authoring page never cross-links the bulk MGIR alternative — its Workflow + See-also teach only the imperative drip, so an agent that lands on material.authoring (not its parent material) does ~24 calls where material.compile_mgir does one"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [material, material-authoring, mgir, compile_mgir, workflow, see-also, discoverability, docs]
---

# `material.authoring` doesn't cross-link the one-call bulk MGIR path, so an agent on that page does the per-node/per-wire drip

The MCP already ships a **one-call bulk material-graph authoring path** —
`material.compile_mgir` (Material Graph IR), shipped DONE under
`F-mgir-material-graph-ir`, *"a text-based bulk authoring … path for material
and material-function assets."* It compiles an entire graph — every expression,
every wire, plus layout via `runLayout:true` (default) — from one text body in a
single call. The **parent** `material` namespace page already surfaces it well:
`docs/wiki-src/material.md` names `material.compile_mgir` / `material.decompile_mgir`
in its prelude (L3), its "How to use" section (L7), and its See-also (L23), and a
dedicated topic page `material.mgir.md` documents the full syntax. So the bulk
path is **not** globally undiscovered — an agent that navigates the `material`
parent finds it.

The narrow residual gap is on the **`material.authoring`** sub-namespace page
specifically. An agent that lands directly on `material.authoring` (the
convenience layer it is steered to for "normal material work") never sees the
bulk alternative there:

- The **Workflow section** (`docs/wiki-src/material.authoring.md` L5-18) teaches
  **only the imperative recipe**: "Create … Add scalar/vector/math/mask nodes;
  keep returned node ids … Connect … Call `compile_material` once at the end." It
  has no "for a large graph, prefer one `material.compile_mgir` call" note.
- The **`## See also`** block (L64-68) — which *does* render on the
  `material.authoring` namespace page — links `material.graph`, `blueprint.graph`,
  and `niagara`, but **omits `material.mgir`**, the one cross-link that would put
  the bulk path in front of an agent reading this page.
- MGIR's only mention on the page is the `### compile_material` per-method
  Cross-ref trailer (L85), which per the wiki rules does NOT render on the
  namespace page — it only surfaces once an agent already calls
  `material.authoring.compile_material` directly, i.e. after they've chosen the
  imperative path.

So an agent that enters at `material.authoring` (rather than its parent
`material`) and follows its Workflow does the full imperative drip even for a
large graph the one-call path was built to win.

## Why it's process friction (clean outcome, ~24 calls for what MGIR does in 1)

The task finished clean with **zero retries** — the friction note is genuinely
"none" for the imperative path (the "Common target pins" table gave every input
name first-try). But the *shape* of the work is the friction: a 10-node
height-fog master material was authored with **~24 imperative execute calls** —
1 `create_material`, 8 `add_*` node calls (WorldPosition, ComponentMask, 2×
scalar param, Subtract, Divide, Clamp, Lerp, 2× vector param), **12
`connect_nodes`**, 1 `auto_layout`, 1 `compile_material` — plus 12 wiki-nav reads
up front to learn the imperative surface. The same graph is one
`material.compile_mgir` body (the IR expresses nodes + wires + layout together),
which is exactly the call MGIR was built to replace the node-add chain with.

This is the "N calls where a single batch should exist" friction — except the
batch **does exist and is DONE**; it simply isn't cross-linked on the
`material.authoring` page the agent read, so an agent that entered there (instead
of the parent `material` page that does surface it) never points itself at it.
The cost recurs on **every** multi-node `material.authoring` build that starts
from this sub-page (this audit is one of a long run of clean stylized-material
builds — see the `E-material-pin-input-type-undiscoverable` #4/#5/#6 history, all
imperative-drip builds of 16-38 calls, none of which reached for MGIR either).

Friction note verbatim: *"none — the wiki pin table (material.authoring.md
'Common target pins') gave exact input names … so every call landed first try
with no retries or fallbacks."* — i.e. the imperative path was smooth; the
process cost is that it was the *only* path the docs steered the agent onto.

This is distinct from the `E-material-pin-input-type-undiscoverable` family
(those are about pin name/type discoverability *within* the imperative path —
making each `connect_nodes` land first-try). This ticket is one level up: that
the imperative path is taken at all for a large graph, because the one-call bulk
alternative is not surfaced where the agent decides how to build.

## Fix (docs-only — the capability already ships, and the parent page already advertises it)

Scope is narrow: surface the bulk alternative on the `material.authoring`
**namespace page** (which the parent `material` already does, but this sub-page
does not). In `docs/wiki-src/material.authoring.md`, both edits land in `##`
sections that render on the namespace page (per the wiki rules):

- Add a one-line note at the end of the **Workflow section (L5-18)**: for a graph
  of more than a few nodes, prefer one `material.compile_mgir` text-IR call over
  the imperative `add_*` + `connect_nodes` chain — it authors all nodes, wires,
  and layout in a single call; the imperative surface is best for small edits or
  touching an existing graph. Link `material.mgir`.
- Add `material.mgir` to the **`## See also`** block (L64-68), alongside the
  existing `material.graph` / `blueprint.graph` / `niagara` links — the one
  cross-link that puts the bulk path in front of an agent on this page.

This is the same "publish the alternative on the page the caller is already on"
remedy argued in `E-material-pin-input-type-undiscoverable` #4, applied one level
up (which path to take, not which pin name to use).

No code change — `material.compile_mgir` / `material.decompile_mgir` /
`material.authoring.auto_layout` all already exist and are verified, and
`material.mgir.md` (the link target) already exists. Page to improve:
`docs/wiki-src/material.authoring.md`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `material.authoring.add_world_position` M_WorldHeightFog task (focus
  `material.authoring.add_world_position`, namespace `material.authoring`,
  outcome `clean`; friction note "none", judge filed nothing). The build was a
  10-node height-fog master material (WorldPosition → ComponentMask(B/Z) →
  Subtract(FogStartHeight) → Divide(FogHeight) → Clamp → Lerp(GroundColor→
  FogColor) into BaseColor, gradient into EmissiveColor) authored with ~24
  imperative execute calls (1 create + 8 add_* + 12 connect_nodes + auto_layout
  + compile_material) preceded by 12 wiki-nav reads, all `ok`, no retries.
  Distinct PROCESS angle from the clean outcome: the MCP already ships
  `material.compile_mgir` (DONE per `F-mgir-material-graph-ir`) — a one-call
  bulk text-IR path that authors the whole graph (nodes + wires + layout) in a
  single call, positioned in the wiki (L83) as the alternative "for bulk MGIR
  text-IR authoring instead of imperative node-add calls." But the
  `material.authoring` overview (L3) and Workflow (L5-18) teach ONLY the
  imperative recipe; MGIR appears solely in the last "Cross-ref:" line of the
  `### compile_material` per-method section, never at the build-decision point or
  on the namespace index / `add_*` pages. So a careful agent following the
  documented Workflow does the full imperative drip for a large graph that MGIR
  was built to do in one call — the "N calls where a single batch exists but is
  undiscovered" friction. Recurs on every multi-node material.authoring build
  (cf. `E-material-pin-input-type-undiscoverable` #4/#5/#6, all 16-38-call
  imperative drips, none reaching for MGIR). Distinct from that pin-name family
  (which makes each imperative `connect_nodes` land first-try); this is one level
  up — that the imperative path is chosen at all because the bulk alternative
  isn't surfaced where the agent decides how to build. Docs-only fix (capability
  already ships): add a "prefer one `material.compile_mgir` call for graphs over
  a few nodes" clause to the overview/Workflow of
  `docs/wiki-src/material.authoring.md` and optionally cross-link MGIR from the
  per-method `connect_nodes`/`add_*` overlay sections. Severity Low — recoverable,
  never blocked; the cost is the per-build call-count overhead, not a failure.
- `#2-reworded-and-docs-fix` `IN-REVIEW` developer — Reworded first: the
  original framing ("undiscovered", "exactly one place / L83", "not on the
  namespace index") was factually overstated. The bulk path is already surfaced
  on the PARENT `material` namespace page in three root-index-visible places
  (`docs/wiki-src/material.md` prelude L3, "How to use" L7, See-also L23) plus a
  dedicated `material.mgir.md` topic page — so it is NOT globally undiscovered.
  Narrowed the title/body/Fix to the real residual gap: the `material.authoring`
  SUB-page (the convenience layer an agent is steered to for "normal material
  work") never cross-links MGIR — its Workflow (L5-18) teaches only the
  imperative recipe, its `## See also` (L64-68) lists `material.graph` /
  `blueprint.graph` / `niagara` but omits `material.mgir`, and MGIR's only mention
  is the `### compile_material` H3 Cross-ref trailer (L85, was cited as L83 — file
  grew 2 lines) which per the wiki rules does NOT render on the namespace page.
  Then implemented the narrow docs-only fix in `Docs/wiki-src/material.authoring.md`:
  (a) a one-line Workflow note ("for a graph of more than a few nodes, prefer one
  `call("material.compile_mgir")` text-IR call over the imperative
  `add_*` + `connect_nodes` drip … see `material.mgir`"), and (b) a `material.mgir`
  bullet added to the `## See also` block. Both edits land in `##` sections, so
  they render onto the `material.authoring` namespace page via FWikiOverlay (the
  H3 Cross-ref did not). No code change — `material.compile_mgir` /
  `material.decompile_mgir` / `material.authoring.auto_layout` already exist and
  are verified, and `material.mgir.md` (the link target) already exists. Test:
  added `FWikiHandlerMaterialAuthoringSurfacesBulkMgirTest` to
  `Source/PinWright/Private/Tests/Infra/TestWikiHandler.cpp` — renders the
  `material.authoring` namespace page via the production `WikiHandler::RenderPage`
  and asserts on two overlay-exclusive markers (`material.compile_mgir`, which is
  registered under the `material` category so it is absent from this page's auto
  `## Methods` index; and the `material.mgir.md` See-also link text). Reverting
  either edit drops these markers — the only other MGIR reference is the
  non-rendering `### compile_material` H3 Cross-ref — and the assertions fail.
  Files: `Docs/wiki-src/material.authoring.md`,
  `Source/PinWright/Private/Tests/Infra/TestWikiHandler.cpp`. Distinct from the
  IN-REVIEW `E-material-pin-input-type-undiscoverable` (pin name/type within the
  imperative path) — same page, non-overlapping edits (which-path-to-take vs.
  which-pin-name-to-use).
- `#3-parallel-reword-and-docs-fix` `IN-REVIEW` developer (concurrent fix host, merged on rebase) — Reworded then implemented.
  Reword (to match reality per the adversarial review): (a) dropped the
  "undiscovered" overclaim — MGIR IS documented at the namespace level on the
  parent `material.md` (`:3,7,23`) and its own `material.mgir.md` topic page; the
  real, narrow gap is that the `material.authoring` *Workflow* (the page an agent
  reads to learn how to build) never steers to it at the build-decision point;
  (b) corrected the citation L83→L85 (the Cross-ref trailer of the
  `### material.authoring.compile_material` H3, which per the wiki render rules
  does NOT render on the namespace page, hence invisible at the decision point);
  (c) dropped the secondary remedy ("cross-link MGIR from the per-method
  `connect_nodes`/`add_*` overlay sections") — those overlay H3 sections do not
  exist in `material.authoring.md` (only `compile_material`, `add_if`, `add_switch`,
  `set_texture_sample_texture`, `create_*`, `add_function_input` are hand-authored),
  and that "echo/link onto per-method pages" mechanism is already owned by the
  IN-REVIEW `E-material-pin-input-type-undiscoverable` #4/#5/#6 for this same page.
  Fix (docs-only — capability already ships): added a one-clause steer to the top
  of the `## Workflow` section of `Docs/wiki-src/material.authoring.md` —
  "for a graph of more than a few nodes, prefer one `call("material.compile_mgir",
  ...)` text-IR call over the imperative `add_*` + `connect_nodes` chain … it
  authors every node, every wire, and layout (`runLayout` defaults true) from a
  single text body … reach for MGIR when building a multi-node graph from scratch.
  See `material.mgir` for the syntax." Lands in the prelude `## Workflow` section
  (above the first `### ` H3) so it renders on the `material.authoring` namespace
  page. Test: added `FMaterialAuthoringMgirWorkflowDocTest` to
  `Source/PinWright/Private/Tests/Infra/TestMaterialAuthoringMgirWorkflowDocs.cpp`
  — renders the `material.authoring` namespace page via the production
  `WikiHandler::RenderPage` and asserts overlay-exclusive markers
  (`material.compile_mgir` present on the namespace page, the "more than a few
  nodes / multi-node graph from scratch" steer alongside `connect_nodes`, and the
  `material.mgir` link); reverting the Workflow clause drops these markers and
  fails the assertions (the auto-generated method index never names a
  "prefer compile_mgir over the imperative chain" steer). Files:
  `Docs/wiki-src/material.authoring.md`,
  `Source/PinWright/Private/Tests/Infra/TestMaterialAuthoringMgirWorkflowDocs.cpp`.
- `#4-additional-asset-namespace-entry-point` `IN-REVIEW` reporter — Additional
  evidence that the same "imperative drip, MGIR bulk path not surfaced at the
  build-decision point" friction recurs from a **different entry page** than the
  `#1`-`#3` fix touches: the **`asset`** namespace, not `material.authoring`.
  Struggle audit of a clean `asset.add_material_node` task (focus
  `asset.add_material_node`, namespace `asset`, outcome `tool_bug` — the judge
  filed `B-material-stats-instruction-count-hardcoded` for the `instructionCount:-1`,
  unrelated to this; friction note verbatim "none - all calls succeeded on first
  try"). The story built a small `M_TintedFloor` master material entirely through
  the legacy `asset.*` material verbs: `asset.create_material`, then 3×
  `asset.add_material_node` (Constant3Vector tint, scalar Constant brightness,
  Multiply), 2× `asset.connect_material_pins` (A←0, B←1), `asset.add_material_parameter`
  (Roughness), `asset.save`, then `asset.get_material_node_details` +
  `asset.get_material_stats` readbacks — the per-node/per-wire imperative drip,
  preceded by 9 wiki-nav reads of the `asset.*` per-method pages. The whole
  node+wire graph (3 nodes + 2 connections) is one `material.compile_mgir` text-IR
  body — the call MGIR was built to replace this chain with. But the agent
  entered through `asset.*` and never saw MGIR: it read `wiki/asset.add_material_node.md`,
  `wiki/asset.connect_material_pins.md`, etc. — none of which point at the bulk
  path. The `#1`-`#3` fix landed only on `Docs/wiki-src/material.authoring.md`'s
  Workflow/See-also, which an `asset.*` agent never reads. The `asset.md` namespace
  page DOES steer legacy `asset.*` material callers to `material.authoring` as the
  canonical surface (L7) — but that steer names only the imperative
  `material.authoring` layer; it does **not** mention the one-call
  `material.compile_mgir` bulk path, and `asset.md`'s only MGIR mention is in the
  dump/`decompile_mgir` context (L83/L195), not at the authoring-decision point.
  So an `asset`-namespace agent gets the imperative drip with no MGIR pointer at
  any of the pages it reads. **Additional page(s) to improve** (same docs-only
  remedy, one level wider than `#2`/`#3`): extend the `asset.md` material-creation
  bullet (L7) — where it already redirects to `material.authoring` — to add "for a
  multi-node graph from scratch, prefer one `material.compile_mgir` text-IR call
  over the imperative `add_material_node` + `connect_material_pins` chain; see
  `material.mgir`", and optionally echo that steer onto the
  `asset.add_material_node` / `asset.connect_material_pins` per-method overlay
  pages. Reinforces that the residual gap is "surface the bulk alternative on the
  page the caller actually entered through" — and there is more than one such
  entry page (`material.authoring`, now `asset`). Severity stays Low — recoverable,
  never blocked; the cost is the per-build call-count overhead (here 5 imperative
  authoring calls for what MGIR does in 1), not a failure.
