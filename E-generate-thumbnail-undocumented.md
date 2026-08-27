---
id: E-generate-thumbnail-undocumented
title: "asset.generate_thumbnail has no wiki page — output format + inline-mode contract undocumented"
status: DONE
severity: Low
category: ergonomic
tags: [docs, asset, thumbnail, discovery]
encounters: 3
lastSeen: 2026-08-27T18:58:26+05:00
---

# asset.generate_thumbnail has no wiki page — output format + inline-mode contract undocumented

`asset.generate_thumbnail` works (honors `assetPath`, `width`, `height`,
`outputPath` and returns `success:true`), but it has **no section in the
`asset.md` wiki page** — `docs/wiki-src/asset.md` documents `map_references`,
`dump`, `dump_folder`, `list`, `fixup_redirectors`, etc. but not
`generate_thumbnail`. The only documentation a caller sees is the inline
tool-schema blurb for `outputPath` ("File path to save thumbnail to disk"),
which says nothing about two contract points the caller must reason about:

1. **What image format `outputPath` produces.** There is no format field and
   no statement of which extensions are honored, so a caller naturally infers
   the format from the extension they pass (`Foo.png` -> PNG). In this task
   that inference was wrong — the bytes are JPEG (tracked as the separate tool
   bug `B-thumbnail-png-writes-jpeg`). Whether the eventual fix is "encode by
   extension" or "always JPEG", the wiki page should state the actual format
   contract so the caller doesn't have to discover it by `od`-ing the file.

2. **What the no-`outputPath` inline mode returns.** `outputPath` is optional;
   omitting it returns only `success`+`width`+`height` with no image bytes and
   no temp path. That is within the documented contract (no field promises
   returned bytes), but nothing on a wiki page tells the caller that inline
   mode yields nothing consumable — so a caller reasonably expects bytes/temp
   path and wastes a probe call discovering otherwise.

This is a pure discovery/usage gap (the RPC succeeded every time), distinct
from the encoder bug: even after that bug is fixed, the missing page leaves the
format and inline-mode contracts undocumented.

**What it should do:** add an `### asset.generate_thumbnail` section to
`docs/wiki-src/asset.md` stating (a) params `assetPath`/`width`/`height`/
optional `outputPath`, (b) the actual on-disk image format and which
extensions are honored, and (c) that omitting `outputPath` returns only
metadata (`success`/`width`/`height`), not image bytes or a temp path — so
callers know on-disk export is the only way to obtain pixels.

**Wiki page to improve:** `docs/wiki-src/asset.md` (add a
`### asset.generate_thumbnail` overlay section).

## Evidence

From this task's friction note: "Discovery and all RPCs were smooth with no
retries" — yet two assumptions baked into the story (PNG-from-`.png`
extension; inline call returns "image bytes or a temp path") were both wrong,
because nothing documented the real contract. Call counts: 5 successful
`asset.generate_thumbnail` calls (3x 512x512 on-disk, 1x 1024x768 hero, 1x
no-outputPath inline). The inline call "returned only success+dimensions with
no image bytes and no resolvable temp path"; the on-disk calls produced
JPEG-in-`.png` files. The format mismatch itself is the tool bug
`B-thumbnail-png-writes-jpeg`; this ticket is the documentation gap that let
the wrong-but-natural assumptions stand unchallenged.

## Encounter 2026-08-27 — the parameters are documented; the documented behaviour is wrong for two of them (re-open candidate)

Recorded here as evidence, **not** a status change — only the tester workflow closes or reopens a
ticket, and this entry does not claim to be that verification.

`#4-verified-in-generated-wiki` signed this off on the strength of the generated page carrying
"the full parameter surface including `primitive` / `primitiveMesh` / `azimuth` / `elevation` /
`zoom`". That is true and still true. What the Atlantis build (2026-08-27, UE 5.8, this checkout)
found is that for two of those parameters the *documented behaviour does not hold* on specific
inputs, so a caller who reads the page and follows it gets a wrong picture:

- `B-thumbnail-plane-elevation-edge-on` — `elevation` is documented as "camera angle in degrees
  above the horizon", in the `camera.frame_actor` vocabulary the `azimuth` doc invokes. On
  `primitive:"plane"` the value is applied faithfully and renders the plane edge on, because the
  thumbnail plane's attitude is a constant chosen for a pitch-0 camera
  (`ThumbnailHelpers.cpp:380-383`) and is not in that vocabulary at all.
- `B-thumbnail-primitive-ignored-on-instances` — the page carries the UI-domain force-plane caveat
  (this ticket's `#3` put it there). There is a **second** forcing rule it does not carry: a
  material instance whose *base* material has particle-sprite or Niagara usage is force-planed
  whatever `primitive` asks for, while the base material itself renders the requested shape.

Both are filed as their own bug tickets and are behaviour defects, not documentation defects — the
right fix is in the handler, and the page then follows. The docs-only reading of this ticket is
what leaves it as a re-open candidate rather than a re-open: if either bug is fixed by *documenting*
the constraint instead of reporting it, the missing sentences belong on the page this ticket owns.
`encounters` and `lastSeen` bumped; `status` deliberately untouched.

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of an `asset.generate_thumbnail` contact-sheet task (focus `asset.generate_thumbnail`, outcome tool_bug, judge filed `B-thumbnail-png-writes-jpeg`). All 5 thumbnail RPCs succeeded with no retries, but `asset.generate_thumbnail` has no section in `docs/wiki-src/asset.md` — the output image-format contract (extension-honored vs always-JPEG) and the no-`outputPath` inline-mode return (metadata only, no bytes/temp path) are both undocumented, so the task built on two natural-but-wrong assumptions. Distinct docs angle from the encoder bug. Proposing an `### asset.generate_thumbnail` overlay section on `asset.md`.
- `#2-deferred-on-encoder` `OPEN` developer — Deferred via `blockedBy: [B-thumbnail-png-writes-jpeg]`. Verified against current source: handler at `AssetWorkflowHandler.cpp:573` still encodes through `FImageUtils::ThumbnailCompressImageArray` (line 649, always JPEG), `ThumbnailEncodeUtils.h` does not exist in this tree, and `docs/wiki-src/asset.md` has no `### asset.generate_thumbnail` section — so the gap is real and the format is "always JPEG" right now. But the ticket's load-bearing deliverable, "(b) the actual on-disk image format and which extensions are honored", is exactly the contract that B's IN-REVIEW `#2-encode-by-extension` fix flips (`.png`→PNG, `.jpg`/`.jpeg`→JPEG, default PNG). Documenting the format now would ship a wiki section that B's fix invalidates the moment it lands here, forcing immediate rework. The param-surface and inline-mode-returns-metadata points are stable, but not worth splitting a single Low-severity overlay edit to ship twice. Gate releases when B-thumbnail-png-writes-jpeg reaches DONE/WONTFIX; then write the section against whichever format contract actually shipped. Lease released.
- `#3-section-written` `IN-REVIEW` developer — Gate lifted: the encoder fix is now IN-REVIEW in this tree (see `B-thumbnail-png-writes-jpeg` `#3-ported-to-pinwright`), so the format contract is settled and the section can be written against what actually ships. Added `### asset.generate_thumbnail` to `Docs/wiki-src/asset.md` covering both load-bearing points from this ticket: (a) the on-disk format contract — the `outputPath` extension selects the encoder, `.jpg`/`.jpeg` write JPEG and everything else including a missing extension writes PNG, with the real format echoed in the response `format` field; (b) that omitting `outputPath` returns metadata only (`success`/`width`/`height`) with no bytes and no temp path, stated explicitly so nobody spends a probe call discovering it. Also documented the full parameter surface including the new `primitive`/`primitiveMesh`/`azimuth`/`elevation`/`zoom` preview controls, that width/height are a MAXIMUM for some asset types (with `requestedWidth`/`requestedHeight` reported when they differ), the square-aspect/narrow-FOV constraint of the thumbnail camera, the UI-material force-plane caveat, and that the verb neither modifies nor dirties the asset. `blockedBy` dropped.

- `#4-verified-in-generated-wiki` `DONE` tester — Verified where it actually matters: the page a
  caller reaches, not just the overlay source. After an editor restart regenerated the tree,
  `Saved/PinWright/wiki/asset.generate_thumbnail.md` exists and carries both load-bearing points from
  this ticket — the extension-selects-the-encoder contract with the `format` echo, and the explicit
  "omitting `outputPath` returns metadata only, no bytes and no temp path" statement — plus the full
  parameter surface including `primitive` / `primitiveMesh` / `azimuth` / `elevation` / `zoom`, the
  width/height-as-a-MAXIMUM caveat with `requestedWidth`/`requestedHeight`, the square-aspect and
  narrow-FOV constraint, the UI-material force-plane caveat, and the never-dirties guarantee. The
  documented format contract was then confirmed against the running verb rather than taken on trust:
  a `.png` path really produced PNG magic bytes and `format: "png"`, a `.jpg` path really produced
  JPEG SOI and `format: "jpeg"` (see `B-thumbnail-png-writes-jpeg` `#4`), so the page describes what
  ships. Committed as `753a3a4a`.
- `#5-documented-behaviour-wrong-for-plane-and-instances` `DONE` reporter — Encounter 2 -> 3, additional evidence only; status deliberately NOT changed (only the tester workflow closes or reopens). Found on the Atlantis build 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. `#4` signed off on the generated page carrying "the full parameter surface including `primitive` / `primitiveMesh` / `azimuth` / `elevation` / `zoom`", which is accurate — but two of those parameters are now filed as behaviour defects where the documented meaning does not hold: `B-thumbnail-plane-elevation-edge-on` (a documented "above the horizon" `elevation` is applied faithfully to `primitive:"plane"` and renders it edge on, because the thumbnail plane's attitude is a constant chosen for a pitch-0 camera at `ThumbnailHelpers.cpp:380-383` and is not in the `camera.frame_actor` vocabulary the docs invoke) and `B-thumbnail-primitive-ignored-on-instances` (the page carries the UI-domain force-plane caveat this ticket's `#3` added, but not the SECOND forcing rule — a material instance whose base material carries particle-sprite or Niagara usage is force-planed whatever `primitive` asks for, while the base material renders the requested shape). Both are handler defects, not doc defects, so they are filed separately; this is a re-open CANDIDATE note because if either is fixed by documenting the constraint rather than reporting it, the missing sentences land on the page this ticket owns.
