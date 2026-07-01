---
id: E-widget-canvas-slot-no-padding-docs
title: "widget.slot-layout overlay documents box-slot Padding and CanvasPanelSlot LayoutData separately, but never states a CanvasPanelSlot has no Padding field — so 'pad a box that sits on a canvas' has no direct answer and forces a get_class_properties probe"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, widget, slot-layout, canvas, padding, CanvasPanelSlot, VerticalBoxSlot, discovery, wiki]
encounters: 1
lastSeen: 2026-06-24T06:10:13Z
---

# `widget.slot-layout` never says a `CanvasPanelSlot` has no `Padding` field — the "pad a canvas-hosted box" intent has no recipe

`docs/wiki-src/widget.slot-layout.md` documents the two slot models in separate
sections:

- **Canvas panel slot layout** (`widget.slot-layout.md:5-65`) — the
  `CanvasPanelSlot.LayoutData` struct: `Offsets` / `Anchors` / `Alignment`,
  anchor-mode-dependent offsets, the default 100×30 trap, centering via
  `Alignment=(0.5,0.5)`. It says "Add non-zero offsets for a margin" but never
  the FMargin word `Padding`.
- **Box slot layout** (`widget.slot-layout.md:67-75`) — children of
  `VerticalBox` / `HorizontalBox` use `Padding` (FMargin),
  `HorizontalAlignment` / `VerticalAlignment`, `Size`.

What the page never states explicitly: a widget hosted **on a CanvasPanel** has
a `CanvasPanelSlot`, which UMG gives **no `Padding` field at all** — only
`LayoutData`. `Padding` exists only on box slots (`VerticalBoxSlot` /
`HorizontalBoxSlot`). So a perfectly natural intent — "give this VerticalBox
some padding and center it in the canvas" — has no direct recipe: padding cannot
live on the box's own (canvas) slot; it must be expressed either as `LayoutData`
offsets/alignment on the canvas slot, or as `Padding` on the box's *children's*
box slots. The two sections sit adjacent but the page never bridges them with
the one sentence that says "canvas slots have no Padding; pad via offsets or via
child box slots."

This is a docs-discovery gap (the tools work; the model of which slot owns
`Padding` is unclear), so an agent confirms it the expensive way — by probing
`widget.get_class_properties { classes:[CanvasPanelSlot, VerticalBoxSlot] }` and
reading that `CanvasPanelSlot` exposes only `LayoutData` while `VerticalBoxSlot`
exposes `Padding` — before it can author the layout.

## Evidence (this task)

Struggle audit of the WBP_PauseMenu UMG layout task (namespace `widget`,
outcome `clean`). The story asked: "Give the VerticalBox some padding and center
it in the canvas." The VerticalBox sits on the root CanvasPanel, so its slot is
a `CanvasPanelSlot` (no `Padding`). The agent had to probe
`widget.get_class_properties` to confirm this, then express padding on the box's
children (`VerticalBoxSlot.Padding`: 20 around the title, 8 between buttons) plus
`LayoutData` centering on the box's canvas slot.

Friction note (verbatim): *"the success check says 'the VerticalBox has non-zero
Slot padding,' but a VerticalBox sitting on a CanvasPanel has a CanvasPanelSlot,
which UMG gives no Padding field (only LayoutData) — confirmed via
get_class_properties — so padding had to be expressed on the box's children
(VerticalBoxSlot.Padding: 20 around the title, 8 between buttons) plus LayoutData
centering; that wording mismatch between the check and how UMG models canvas
slots is the only real snag."*

(The "success check wording" half of the note is a test-harness criterion, not a
tool defect — the tool-side, durable takeaway is the docs gap: nothing in the
overlay tells the caller that a canvas slot has no Padding, so the
"pad a canvas-hosted box" recipe is missing.)

## What it should do / how to fix (docs-first, NAMES the overlay page)

Improve `docs/wiki-src/widget.slot-layout.md`. The overlay edit is a downstream
wiki process; naming the page and the change is the deliverable:

- In the **Canvas panel slot layout** section, add one explicit sentence: a
  `CanvasPanelSlot` has **no `Padding` field** — only `LayoutData`
  (`Offsets` / `Anchors` / `Alignment`). `Padding` (FMargin) exists only on box
  slots (`VerticalBoxSlot` / `HorizontalBoxSlot`).
- Add a short "padding a box that sits on a canvas" recipe bridging the two
  sections: to inset a canvas-hosted VerticalBox, either (a) use non-zero
  `Offsets` / `Alignment=(0.5,0.5)` on its `CanvasPanelSlot.LayoutData` for the
  outer margin + centering, and/or (b) set `Padding` on the box's *children's*
  `VerticalBoxSlot`s for internal spacing — because the box's own (canvas) slot
  carries none.
- Optionally cross-link the schema-probe verb
  (`widget.get_class_properties { classes:[CanvasPanelSlot, VerticalBoxSlot] }`)
  as the way to confirm which slot class owns which field — so the recipe is
  self-verifiable without trial-and-error.

**Workaround:** treat `CanvasPanelSlot` as `LayoutData`-only; for a canvas-hosted
box, center/inset via `LayoutData` `Offsets`+`Alignment` and put any internal
`Padding` on the box's child `VerticalBoxSlot`s.

## Not a duplicate of

- `B-widget-set-slot-struct-fails` / `B-configure-slot-behavior-ignores-behavior-and-tags`
  — those are slot-*write* bugs (struct application failures); this is a
  docs-model gap about which slot class owns `Padding`, no tool misbehaves.
- `E-widget-describe-slot-truncation` — slot-readback truncation, unrelated to
  the canvas-vs-box Padding ownership model.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the WBP_PauseMenu UMG
  layout task (namespace `widget`, outcome `clean`, no judge filing). PROCESS
  friction: the "give the VerticalBox some padding and center it in the canvas"
  intent has no direct recipe in `docs/wiki-src/widget.slot-layout.md` — the page
  documents box-slot `Padding` and `CanvasPanelSlot.LayoutData` in adjacent
  sections but never states a `CanvasPanelSlot` has **no** `Padding` field, so a
  box hosted on a canvas can't be padded on its own slot. The agent had to probe
  `widget.get_class_properties { classes:[CanvasPanelSlot, VerticalBoxSlot] }` to
  confirm and then express padding on child `VerticalBoxSlot`s + `LayoutData`
  centering. Friction note (verbatim): *"a VerticalBox sitting on a CanvasPanel
  has a CanvasPanelSlot, which UMG gives no Padding field (only LayoutData) —
  confirmed via get_class_properties — so padding had to be expressed on the
  box's children ... that wording mismatch ... is the only real snag."* Fix
  (docs-only): add to `widget.slot-layout.md` the explicit "canvas slots have no
  Padding" statement plus a "pad a canvas-hosted box" recipe bridging the
  LayoutData-vs-box-Padding sections.
