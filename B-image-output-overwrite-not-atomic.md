---
id: B-image-output-overwrite-not-atomic
title: "Image and thumbnail writers replace final paths directly; multi-file failures leave mixed old/new output sets"
status: WONTFIX
severity: Low
category: bug
tags: [image, thumbnail, overwrite, atomicity, partial-output, filesystem]
---

# Image outputs are written directly over their final files

## What's wrong

The shared PNG writer calls `SaveArrayToFile` on the final path
(`Handlers/Image/ImageOps.cpp:151-182`). The callers preflight overwrite policy but do not stage or
restore content:

- `image.annotate` writes directly to its single destination (`ImageAnnotateHandler.cpp:287-291`,
  `:797`).
- `image.compare` writes two outputs sequentially (`ImageCompareHandler.cpp:269-290`); if the
  second fails, `partialOutputs` admits the first remains, but overwrite mode has already destroyed
  its previous bytes.
- `image.tile` writes tiles one by one and then writes the manifest directly
  (`ImageTileHandler.cpp:181-258`, `:306-312`), so failure leaves a mixed old/new tile set and a
  stale or absent manifest.
- `asset.generate_thumbnail` has no overwrite parameter at all and saves encoded bytes directly
  to caller-supplied `outputPath` (`AssetWorkflowHandler.cpp:1004-1006`).

These are regenerable artifacts and most image verbs expose overwrite intent, so impact is Medium;
the bug is that an acknowledged failure cannot restore the previous coherent output set.

## What it should do

Stage the complete output set beneath a unique temporary root, then atomically replace finals only
after every encode/write succeeds. Back up existing files for rollback. Thumbnail generation must
add an explicit overwrite policy (default false).

## Workaround

Write to a new output directory/name with overwrite disabled, then move the verified complete set
into place outside PinWright.

## History
- `#1-pattern-scan` `OPEN` reporter — Grouped because all four routes share direct-final filesystem replacement. Source only; no editor, build, test, or RPC run.
- `#2-re-rated` `OPEN` triage — Severity Medium -> Low. The damage occurs only when a multi-file write fails mid-set (a rare edge path), the outputs are regenerable images rather than asset data, and `image.compare` already reports `partialOutputs`; Medium impact bumped down one for reach.
- `#3-stale-sweep-yagni` `WONTFIX` developer — Still accurate at plugin HEAD `10212ee4` (`Handlers/Image/ImageOps.cpp:175` and `Handlers/Asset/AssetWorkflowHandler.cpp:1450` save straight to the final path), but this came from a source-only pattern scan with no observed failure. The damage needs a multi-file write to fail partway, the outputs are regenerable images rather than asset data, `image.compare` already reports `partialOutputs`, and the documented workaround (write to a fresh name, then move) is cheap. Staging and rollback for every image writer is speculative hardening.
