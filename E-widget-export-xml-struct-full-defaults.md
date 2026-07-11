---
id: E-widget-export-xml-struct-full-defaults
title: "`widget.export_xml` expands an overridden struct prop to its full defaults, overflowing the display limit on a tiny tree"
status: OPEN
severity: Low
category: ergonomic
tags: [struct-default-subfield-bloat, widget, export-xml]
encounters: 1
lastSeen: 2026-07-11T23:07:46.2163121+03:00
---

# `widget.export_xml` dumps every default subfield of an overridden struct property

When a widget-tree element overrides a **struct-valued** UPROPERTY (here a
TextBlock's `Font` / `FSlateFontInfo`), `widget.export_xml` serializes the
**entire struct including every default subfield** rather than only the
subfields the author actually set. On a freshly-built 6-widget pause menu
(VerticalBox root + a "Paused" TitleText + two Button/Label pairs) where the
only Font overrides were `Size` and `TypefaceFontName`, each TextBlock came
back with a full struct blob shaped like:

```
Font="{FontObject=/Engine/EngineFonts/Roboto.Roboto,FontMaterial=,OutlineSettings={OutlineSize=0.0,bMiteredCorners=false,bSeparateFillAlpha=false,bApplyOutlineToDropShadows=false,OutlineMaterial=,OutlineColor={R=0.0,G=0.0,B=0.0,A=1.0}},TypefaceFontName=...,Size=...,LetterSpacing=0.0,SkewAmount=0.0,bForceMonospaced=false,bMaterialIsStencil=false,MonospacedWidth=1.0,FontName=None,Hinting=Default}"
```

i.e. `FontObject`, `FontMaterial`, the whole `OutlineSettings` sub-struct,
`LetterSpacing`, `SkewAmount`, `bForceMonospaced`, `bMaterialIsStencil`,
`MonospacedWidth`, `FontName`, `Hinting` — all emitted at their defaults even
though none were touched. Three near-identical Font blobs are the bulk of the
response.

## What's awkward
Those default-valued subfields pushed a trivial hand-authored tree from ~1KB of
meaningful markup to **10,238 chars**, tripping the 10,000-char display
threshold, spilling the payload to
`Saved/PinWright/HttpResponses/...json`, and forcing one extra out-of-band
`Read` to recover the tree that the readback was supposed to hand back inline.
Geometry emission was already off (`Geom.source=off`), so it was not the cause.

## What it should do
Emit only the **overridden subfields** of a changed struct (or a compact
struct rendering), so small hand-authored trees stay inline and legible without
a file round-trip. The wiki bills the method as returning markup "with all
overridden properties", which reads as subfield-level minimalism, not a
full-struct default dump.

## Relationship to the existing token-limit ticket
This is a **distinct root cause** from `E-widget-export-xml-token-limit`
(DONE). That ticket targets the `Slot="{...}"` attribute bloat — the
`Parent={Slots=...}` sibling chain and the `Content={...}` block — and its
shipped `compact` / `omit_slot_chain` fix works by skipping the raw widget
`Slot` property. The Font blob here is a **top-level attribute on the TextBlock
element itself in the widget tree**, not inside a Slot, so the shipped
Slot-chain elision does not remove it. On this shallow tree the Slot chain is
minimal and the full-struct Font blobs dominate the overflow — a verbosity
source the compact fix leaves untouched. A fix here (per-struct subfield
minimization) would also clean up the CDO-default noise the token-limit ticket
notes inside `Content`.

## Evidence
- Single `widget.export_xml` call at the tree-verify step returned
  `{"outputTooLong":true}` at 10,238 chars vs the 10,000 display threshold; the
  elided payload is dominated by three near-identical full `FSlateFontInfo`
  struct blobs. Source: CallAnalyzer trace (14-call clean attempt, `plan_divergence: none`).
- Agent friction note (self-reported): "the export_xml response exceeded the
  display limit and was written to a file I then read from disk."
- Judge disposition classified the overflow itself as the known DONE behavior of
  the token-limit ticket (compact opt-in confirmed working); it did not address
  the struct-subfield expansion as a separate verbosity source.

severity rationale: impact=response-spill that only forces a Read (task
succeeded in one Read, no retry, no wrong data) x reach=rare (only when a
struct-valued widget prop like Font is overridden on a small tree that would
otherwise fit inline) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — `widget.export_xml` on a fresh 6-widget pause menu overflowed at 10,238 chars (vs 10,000), spilled to a fallback file and forced one extra Read; overflow bulk is three full `FSlateFontInfo` struct blobs emitted with every default subfield though only `Size`/`TypefaceFontName` were overridden. Distinct root cause from `E-widget-export-xml-token-limit` (Slot-chain bloat, DONE) — the Font blob is a tree-element attribute the shipped Slot-chain compact fix does not elide. Proposes emitting only overridden struct subfields (or a compact struct rendering).
