---
id: B-animation-provenance-misclassified
title: "Animation ownership refusals use the malformed-value diagnostic code"
status: IN-REVIEW
severity: Medium
category: bug
tags: [pwanim, provenance, diagnostics, ownership]
---

# Animation ownership refusals look like malformed source values

When animation compilation targets an asset stamped by a different source, creation is
refused unless takeover is explicitly permitted. The compiler previously converted that
ownership refusal to the generic `PWSRC_BAD_VALUE` code, which is also used for genuine
source value and readback errors. Callers therefore could not reliably choose between
fixing malformed source and granting an intentional takeover.

**Impact:** callers receive the wrong recovery instruction for a valid source that targets
an asset owned by another source.

**Fix:** emit `PWANIM_ASSET_PROVENANCE_CONFLICT` for ownership refusals and preserve
`PWSRC_BAD_VALUE` for genuine malformed-value failures. The ownership message names the
source currently stamped on the asset.

## History
- `#1-code-collision-reproduced` `OPEN` reporter — Reproduced an animation target stamped by a different source. Asset creation returned the ownership refusal, but `PwAnimCompiler` mapped it to `PWSRC_BAD_VALUE`, the same code returned by malformed timebase values.
- `#2-split-provenance-diagnostic` `IN-REVIEW` developer — Added `PWANIM_ASSET_PROVENANCE_CONFLICT` and mapped animation asset-ownership refusals to it while retaining the existing owner-naming message. Added one two-direction test: the ownership case must emit the new code and not `PWSRC_BAD_VALUE`; a malformed zero-frame timebase must emit `PWSRC_BAD_VALUE` and not the provenance code. Full build: `Result: Succeeded`; Animation.Compiler passed 6/6 and the diagnostic catalog passed 1/1, with zero failures or skips. Ticket remains `IN-REVIEW` for independent tester disposition.
