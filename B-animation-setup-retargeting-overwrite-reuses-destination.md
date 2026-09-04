---
id: B-animation-setup-retargeting-overwrite-reuses-destination
title: "animation.setup_retargeting overwrite=true reuses the old destination instead of replacing it with the source animation"
status: IN-REVIEW
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

## Fix

**Verdict: TRUE.** The overwrite branch reused the loaded destination and never duplicated the selected source. `AnimationHandler.cpp` now creates a uniquely named source-derived stage, assigns the target Skeleton, verifies and saves that stage to disk before touching the old destination, then transactionally publishes and force-saves it. The old object is deleted only after the new destination is durable; rename/save failures restore the original identity and report `RENAME_FAILED` or `SAVE_FAILED` with the observed disk state.

Files: `Source/PinWright/Private/Handlers/Animation/AnimationHandler.cpp`, `Source/PinWright/Private/Tests/Animation/TestSetupRetargetingContract.cpp`, and `Docs/wiki-src/animation.md`. Regression ID: `PinWright.animation.setup_retargeting.OverwriteStagesSourceBeforeReplacement`.

Deliberately unchanged: this legacy verb still does not perform IK retargeting; the response discloses that separate contract instead of calling the output retargeted.

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed the existing-destination overwrite branch never duplicates the chosen source and instead changes/marks dirty the old destination; no editor, build, or test was run.
- `#2-staged-overwrite-replacement` `IN-REVIEW` developer — Changed `AnimationHandler.cpp` to validate a source-derived staged copy before replacing an existing destination, retain a rollback backup during publish, and report the non-IK operation honestly; added structural regression coverage under `Tests/Animation`.
- `#3-verifier-hardening` `IN-REVIEW` developer — Made staging durable before publication, delayed old-object deletion until the replacement save succeeds, added tick-safety classification for the GC-capable verb, and replaced source-grep coverage with success and rollback handler tests.
