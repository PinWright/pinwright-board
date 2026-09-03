---
id: B-animation-setup-retargeting-overwrite-reuses-destination
title: "animation.setup_retargeting overwrite=true reuses the old destination instead of replacing it with the source animation"
status: OPEN
severity: High
category: bug
tags: [animation, retargeting, overwrite, collision, corruption, false-success]
---

# `overwrite=true` mutates stale destination data and calls it retargeted

When the destination exists, the handler only skips for `overwrite=false` (`Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/AnimationHandler.cpp:1069-1076`). With `overwrite=true` it falls through without calling `DuplicateAsset`, because duplication is confined to the `else if` for a nonexistent destination (`:1077-1083`). It then loads the pre-existing destination, changes its Skeleton pointer, marks it dirty, and returns that path in `retargetedAssets` (`:1085-1104`, `:1107-1137`). The selected source sequence is never copied into the destination.

A caller trying to replace stale output B with newly edited source A instead leaves B's old tracks under the target Skeleton, marks that package dirty, and gets success. This destroys B's prior correct in-memory Skeleton binding and leaves stale/incompatible animation data ready for the next save—the exact opposite of overwrite semantics.

Use the catalog's safe replacement shape: produce the genuinely retargeted source in a staged temporary asset, validate its target Skeleton/tracks, then atomically swap it into the requested destination. On any failure keep the old destination untouched and report the failed phase. Add a differential test where source and pre-existing destination have distinct track signatures; overwrite must leave the source-derived signature. Report mark-dirty versus disk persistence honestly.

**Workaround:** delete or rename the destination before calling, and do not trust this verb as an actual retargeter until its sibling defect is fixed.

## Related

- Catalog: `unsafe-output-replacement-or-collision`, `wrong-target-identity-or-fallback`, `terminal-success-before-completion-or-invariant`
- `B-animation-setup-retargeting-does-not-retarget`

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed the existing-destination overwrite branch never duplicates the chosen source and instead changes/marks dirty the old destination; no editor, build, or test was run.
