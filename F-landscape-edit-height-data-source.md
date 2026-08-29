---
id: F-landscape-edit-height-data-source
title: "landscape.edit operation='set' takes heightData ONLY as an inline array — 255,025 numbers for a default landscape — with no file, render-target, texture or function source, so a computed heightfield cannot be installed from an MCP client"
status: OPEN
severity: Medium
category: feature
tags: [landscape, edit, heightData, write-verb-no-content-source, heightfield, import, payload-size, asymmetry]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The write side takes the whole heightfield inline; the read side already concedes that does not fit

`landscape.edit` (`LandscapeHandler.cpp:1626`) is the verb for installing a
heightfield you already have — `landscape.sculpt`'s own summary at `:830` says
"Prefer landscape.edit when you already have a heightfield to install." Its
`operation` parameter (`:1630`) offers `'set'` for exactly that. The height data
has one and only one shape, declared at `:1631` and reproduced verbatim in
`Saved/PinWright/wiki/landscape.edit.md:14`:

> `heightData` (`array`, optional): Row-major uint16 heightmap samples. Required
> for operation='set'; length must match the region size.

`array` is the whole surface. The handler reads it with `TryGetArrayField` at
`:1673-1675`, refuses `'set'` without it at `:1677-1680` (`INVALID_ARGUMENT`,
"heightData array required for 'set' operation"), and converts it element by
element at `:1686-1692`. There is **no `heightDataPath`, no `sourceTexture`, no
`renderTarget`, no `heightmapAsset`, no function/expression source, and no
encoded (base64 / PNG16 / r16) form** — the complete parameter list at `:1628-1634`
is `landscapePath`, `landscapeName`, `operation`, `heightData`, `region`,
`skipFlush`.

For the verb's own documented default landscape — `landscape.create` at `:387`
advertises "8x8 of 63x63 quads (~505x505 vertex resolution)" — a full-extent
`set` is **255,025 numbers in one JSON request**. That is the smallest default
case, not a pathological one.

## The asymmetry is the argument

The read counterpart shipped with truncation built in, because the plugin already
knows raw sample arrays do not fit in a message. `landscape.get_heights`
(`:1840`) declares, at `:1845-1846`:

- `includeSamples` (`boolean`) — "Return the raw row-major uint16 height samples
  (capped by maxSamples). Defaults to **false** — only the region aggregates are
  returned."
- `maxSamples` (`number`) — "Cap on the number of raw samples returned when
  includeSamples=true. Defaults to **4096**."

Parsed at `:1884-1886`; the cap is applied in
`LandscapeHeightStats.cpp:86` (`FMath::Min(MaxSamples, Heights.Num())`), and the
response carries a `samplesTruncated` flag. So the *read* direction: raw samples
are off by default, capped at 4096 by default, and the primary answer is
aggregates instead. The *write* direction on the same heightfield, in the same
handler, one screen apart: the full array inline, no cap, no alternative, and it
is mandatory.

**4096 out, 255,025 in.** The plugin concedes the size problem in one direction
and requires 62x the conceded limit in the other.

The consequence is not that a call fails — it is that the whole class of work
"compute a heightfield, install it" is unreachable *through the verb built for
it*. Every offline heightfield source a caller would actually have — a `.r16`
export, a render target rendered in-editor, a noise/erosion result, a real-world
DEM tile — has to be flattened into JSON numbers to cross the wire.

## Workaround, and why it is not enough to make this Low

Two exist, and the ticket must name both rather than claim "no workaround":

1. **Chunk with `region`.** `region` (`:1632`) restricts the write, so a 505x505
   surface can go in as 64 calls of 4096 samples. This is real and documented, and
   it is why this is a soft blocker. It does **not** reduce the total volume — the
   same 255,025 numbers cross the wire, in 64 pieces — so it converts one
   impossible call into many expensive ones rather than making the task cheap.
2. **`python.execute`** (`PythonExecuteHandler.cpp:116`) runs arbitrary UE Python
   in-process, where the array never crosses the wire at all. This is the same
   correction `F-landscape-height-readback` `#2` applied to its own "Workaround:
   none" claim, and it applies here for the same reason. It is also the reason
   this ticket is not High: the capability is reachable, just not through the
   typed surface, and reaching it means writing engine Python against
   `FLandscapeEditDataInterface` — a source dive, which is exactly what the
   rubric's Medium band describes.

## What it should do

Accept a height source that is not the samples themselves. In rough order of
value against implementation cost:

1. **`heightDataPath`** — a project-relative file of raw little-endian uint16
   (`.r16`, the format UE's own landscape import already reads), plus the
   `region` it covers. This is the one that unblocks every offline producer, and
   the engine-side reader already exists.
2. **`sourceTexture`** — a `UTexture2D` / `UTextureRenderTarget2D` asset path,
   sampled to the region. The plugin can already create and write render targets,
   so this closes the loop entirely in-editor with nothing crossing the wire.
3. **Encoded inline** — base64 of the same uint16 block. ~4x smaller than JSON
   numbers and a much smaller change than either of the above, but it does not
   remove the wire cost, so it is a mitigation rather than a fix.

Whichever lands, it should mirror `get_heights`' honesty: echo the sample count
actually written and the region actually covered, so a partial or mis-sized
source is visible rather than silent.

severity rationale: impact — the rubric's "High or Medium: hard blocker with no
workaround (a stub, a missing verb, or rejecting valid input), so a reasonable
task is impossible" is the right band, and I place it at the **Medium** end
rather than High because the task is not actually impossible: `region` chunking
is documented and works, and `python.execute` reaches the same engine call
directly. That makes it the rubric's Medium proper — "doable, but only via a
documented workaround, a source dive, or many extra calls" — and all three of
those clauses apply literally. Reach: `landscape.edit operation='set'` is not an
almost-every-session method, so I **decline the reach bump-up**; nor is it a rare
edge path (it is the only verb for installing a heightfield, and the one
`landscape.sculpt`'s own docs redirect to), so I **decline the bump-down** to Low.
-> **Medium**.

## Same shape as

- `F-landscape-height-readback` (IN-REVIEW, Low) — **the read-direction sibling,
  and this ticket is its exact inverse.** Its family tag is
  `write-verb-no-content-readback`: a write verb whose content could not be read
  back. This is `write-verb-no-content-source`: the same write verb, whose content
  cannot be *supplied* by anything but an inline array. Its `#2` is what shipped
  `landscape.get_heights` — including the `includeSamples` / `maxSamples`
  truncation this ticket cites as the plugin's own admission that sample arrays do
  not fit in a message. **Its `#1` dedup survey never raised the write-source
  question**, and neither does anything else in the file: I read both history
  entries in full, and every mention of the write direction is used only to
  establish that a *read* was missing or that the read mechanism already existed on
  the write path ("`landscape.edit` only WRITES (`operation='set'`/flatten) — it
  cannot read a region back"; "the write path already reads heights via
  `FLandscapeEditDataInterface::GetHeightData`"). The provenance of the data a
  caller passes *in* is absent from that ticket. Its `#1` survey enumerated the
  landscape files as bounds-refresh, create-geometry, ctx-bypass, error-text and
  param-naming — no height-data-source ticket then, and none now.

## History
- `#1-write-side-has-no-source` `OPEN` reporter — Filed from source, every citation re-derived at HEAD in this checkout; not RPC-replayed in this session. `landscape.edit`'s `heightData` (`LandscapeHandler.cpp:1631`, wiki `landscape.edit.md:14`) is declared `array` and nothing else; the full param list at `:1628-1634` is landscapePath / landscapeName / operation / heightData / region / skipFlush, with no file, texture, render-target, function or encoded source, and `'set'` without the array is refused `INVALID_ARGUMENT` at `:1677-1680`. A full-extent set on the verb's own documented default landscape (8x8 x 63 quads, "~505x505 vertex resolution", `:387`) is 255,025 inline numbers. The argument is the asymmetry inside one handler: the read counterpart `landscape.get_heights` (`:1840`) shipped `includeSamples` defaulting to **false** and `maxSamples` defaulting to **4096** (`:1845-1846`, parsed `:1884-1886`, capped `LandscapeHeightStats.cpp:86`, with a `samplesTruncated` flag) precisely because raw sample arrays do not fit in a response — so the plugin concedes the size problem on the read side while the write side requires 62x the conceded cap with no alternative. Workarounds named rather than denied: `region` chunking (documented, moves the same volume in ~64 calls) and `python.execute` (`PythonExecuteHandler.cpp:116`, the same escape hatch `F-landscape-height-readback` `#2` used to correct its own "Workaround: none"), which is why this is Medium and not High. Cross-linked to `F-landscape-height-readback` (IN-REVIEW, Low) as the inverted sibling; confirmed by reading it in full that its `#1` dedup survey and `#2` implementation note never raise the write-source question — both discuss the write verb only to establish that the read was missing.
