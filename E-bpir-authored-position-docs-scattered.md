---
id: E-bpir-authored-position-docs-scattered
title: "Authored-position-mode BPIR contract is scattered across ~6 wiki pages with no landing/index entry; agents guess a non-existent caller page before the first compile_bpir"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, bpir, authored-position, discoverability, wiki-nav, compile_bpir]
encounters: 2
lastSeen: 2026-06-29T02:57:29Z
---

# Authored-position-mode contract has no single discoverable home

An agent authoring a fully-positioned BPIR body (every node carries `@(x, y)`,
auto-layout suppressed) must assemble the contract from pieces spread across many
BPIR wiki pages before it can make a single confident `blueprint.compile_bpir`
call. The discrete rules a positioned-authoring caller needs —

- coordinate-preservation guarantee (authored `@(x,y)` preserved verbatim, no
  drift/rounding/grid-snap, no extra suffix on lines you didn't position),
- the layout-mode trigger (all-positioned vs mixed vs none) and its
  implicit-visible-helper policy,
- the explicit-`exec -> @label` reconvergence requirement (fall-through across
  label boundaries does not wire),

— live on different pages (`bpir.entry-points.md`, `blueprint.bpir-gotchas.md`,
`blueprint.compile_bpir.md`, `bpir.instructions.md`, `bpir.examples.if-else.md`),
with no "authored-position mode" landing section or index entry that names the
mode and links the parts. There is also no caller-facing `ir-authoring` page,
even though that is an intuitive name an agent reaches for first (the existing
`Docs/ir-authoring.md` from DONE `F-ir-authoring-guide` is an internal
*maintainer* guide for writing new IRs, not served in the caller wiki).

This is **pure discovery friction** (the contract content mostly exists; it is
just not co-located or named), hence Low. The two *correctness* gaps the scatter
exposes are tracked elsewhere and are **out of scope here** — this ticket is the
navigation/landing-page fix only:

- the misleading reconvergence example / silently-dropped fall-through edge →
  `B-bpir-fallthrough-reconverge-dropped` (compiler bug + its own docs note),
- the implicit `UK2Node_Self` / `Break*` helper not being rejected →
  `B-bpir-positioned-implicit-helper-not-rejected`.

## Evidence

From a `blueprint.compile_bpir` authored-position round-trip-equivalence probe
(namespace `blueprint`, transcript `agent-a4f4f72eb42e0d70c.jsonl`, outcome a
filed tool bug). Before the **first** authoring call the agent made **14** wiki
Read/Glob/Grep discovery tool-uses — `index.md`, `blueprint.md`, `bpir.md`,
`bpir.instructions.md`, `bpir.entry-points.md`, `blueprint.compile_bpir.md`,
`bpir.examples.if-else.md`, `blueprint.bpir-gotchas.md`, `blueprint.decompile.md`,
`blueprint.create.md`, `bpir.examples.custom-event-with-params.md`,
`blueprint.graph.md`, plus a `Glob` for `**/*ir*.md` — **including one FAILED
`Read` of a guessed page `wiki/ir-authoring.md`** (returned "File does not
exist"). The authored-position rules and the reconvergence/fall-through semantics
the task needed were split across ~6 of those pages. Partly inherent to an
adversarial BPIR probe (some breadth is expected), which is why this is Low and
scoped to navigation only.

## What it should be

A single discoverable "Authored-position mode" landing section — likely on
`bpir.entry-points.md` (which already hosts the layout-mode table) or
`blueprint.bpir-gotchas.md` — that names the mode and cross-links the three rule
clusters (coordinate preservation, implicit-helper policy, explicit-exec
reconvergence) so a caller reaches the whole contract from one entry point
instead of six. Optionally add a caller-facing `ir-authoring` wiki alias/redirect
(the agent's intuitive page name) pointing at the BPIR entry-points page. Wiki
overlay edits only (downstream wiki process) — no code.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the CallAnalyzer trace of a `blueprint.compile_bpir` authored-position round-trip-equivalence probe (transcript `agent-a4f4f72eb42e0d70c.jsonl`). Discovering the authored-position contract (coords-preserved-verbatim + auto-layout-suppressed + implicit-helper policy + explicit-exec reconvergence) required **14** wiki discovery tool-uses across ~6 BPIR pages before the first authoring call, including a **failed `Read` of a guessed `wiki/ir-authoring.md`** (does not exist; the only `ir-authoring` doc is the internal maintainer guide from DONE `F-ir-authoring-guide`, not a caller wiki page). Proposed: a single named "Authored-position mode" landing/index section co-locating + cross-linking the three rule clusters (likely on `bpir.entry-points.md` or `blueprint.bpir-gotchas.md`), plus an optional caller-facing `ir-authoring` alias/redirect. Navigation/discoverability only — the two correctness gaps the scatter exposes are tracked on `B-bpir-fallthrough-reconverge-dropped` and `B-bpir-positioned-implicit-helper-not-rejected` and are out of scope here. Partly inherent to an adversarial BPIR probe, hence Low. Dedup: rg across board for `authored-position`/`scattered`/`discoverab`/`ir-authoring`/`wiki-nav` found no existing navigation ticket for the authored-position contract (the implicit-helper and fall-through tickets cover content corrections, not co-location); `F-ir-authoring-guide` is a DONE maintainer doc, different surface.
- `#2-cross-task-ir-authoring-glob-recurs` `OPEN` reporter — Cross-task corroboration (struggle-auditor; authored-position node-FORMATTING round-trip-equivalence probe, focus `blueprint.compile_bpir`, transcript `agent-a4dab0f116a59dd4c.jsonl`, fresh Actor BP `/Game/BP_BpirFmtRoundTrip`). The same intuitive-but-nonexistent `ir-authoring` page recurred on an independent task: the agent's first discovery move included a `Glob` for `*ir-authoring*` that **returned no files**, recovered via a `bpir*` glob — the exact "agents reach for a non-existent ir-authoring page first" signal this ticket is about. Before the first `compile_bpir` the attempt read ~14 BPIR wiki pages (`index`, `blueprint.compile_bpir`, `blueprint.decompile`, `bpir.instructions`, `bpir.entry-points`, `blueprint.bpir-gotchas`, `bpir.examples.if-else/simple-beginplay/sequence`, `blueprint.create/compile`, the `blueprint.graph.*` readers, `asset.save`) to assemble the authored-position contract (coords-preserved + auto-layout-suppressed + implicit-helper policy + explicit-exec reconvergence). The CallAnalyzer judged this Attempt's breadth "thorough but targeted ... not a real nav gap" (so this is corroboration on the existing Low ticket, not a new finding), but the recurring failed `ir-authoring` glob across an independent task reinforces the proposed caller-facing `ir-authoring` alias/redirect plus a single named "Authored-position mode" landing section. Dedup: matched this OPEN ticket on rg `ir-authoring`/`authored-position`/`scattered`; appended rather than re-filed.
