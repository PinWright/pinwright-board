---
id: E-generate-thumbnail-undocumented
title: "asset.generate_thumbnail has no wiki page — output format + inline-mode contract undocumented"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, asset, thumbnail, discovery]
blockedBy: [B-thumbnail-png-writes-jpeg]
encounters: 2
lastSeen: 2026-06-24T06:10:12Z
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

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of an `asset.generate_thumbnail` contact-sheet task (focus `asset.generate_thumbnail`, outcome tool_bug, judge filed `B-thumbnail-png-writes-jpeg`). All 5 thumbnail RPCs succeeded with no retries, but `asset.generate_thumbnail` has no section in `docs/wiki-src/asset.md` — the output image-format contract (extension-honored vs always-JPEG) and the no-`outputPath` inline-mode return (metadata only, no bytes/temp path) are both undocumented, so the task built on two natural-but-wrong assumptions. Distinct docs angle from the encoder bug. Proposing an `### asset.generate_thumbnail` overlay section on `asset.md`.
- `#2-deferred-on-encoder` `OPEN` developer — Deferred via `blockedBy: [B-thumbnail-png-writes-jpeg]`. Verified against current source: handler at `AssetWorkflowHandler.cpp:573` still encodes through `FImageUtils::ThumbnailCompressImageArray` (line 649, always JPEG), `ThumbnailEncodeUtils.h` does not exist in this tree, and `docs/wiki-src/asset.md` has no `### asset.generate_thumbnail` section — so the gap is real and the format is "always JPEG" right now. But the ticket's load-bearing deliverable, "(b) the actual on-disk image format and which extensions are honored", is exactly the contract that B's IN-REVIEW `#2-encode-by-extension` fix flips (`.png`→PNG, `.jpg`/`.jpeg`→JPEG, default PNG). Documenting the format now would ship a wiki section that B's fix invalidates the moment it lands here, forcing immediate rework. The param-surface and inline-mode-returns-metadata points are stable, but not worth splitting a single Low-severity overlay edit to ship twice. Gate releases when B-thumbnail-png-writes-jpeg reaches DONE/WONTFIX; then write the section against whichever format contract actually shipped. Lease released.
