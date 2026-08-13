---
id: B-package-fab-decimal-range-false-positive
title: "Package facts gate mistakes numeric ratios for engine ranges"
status: OPEN
severity: Medium
category: bug
tags: [packaging, validation, product-facts]
encounters: 1
lastSeen: 2026-08-13T00:00:00Z
---

# Package facts gate mistakes numeric ratios for engine ranges

The published-number consistency gate treats every dotted decimal range in a
Markdown file as an Unreal Engine compatibility claim. It therefore rejects the
level-design guidance `1.0–1.25 canopy-diameters apart` as an alleged supported
engine range that conflicts with `5.3-5.8`.

This is a packaging hard blocker on valid public prose. Adding a per-file
exception would encode a content accident instead of fixing the classifier.

**Fix:** Require the range to be introduced by `UE` or `Unreal Engine` before
comparing it with the product-facts range. Keep the existing spelling and
per-API exemption behavior for actual engine-version statements.

## History
- `#1-canopy-ratio-false-positive` `OPEN` reporter — Reproduced after the public-text blocklist passed: `package-fab.ps1 -ValidateOnly -KeepStage` rejected `1.0–1.25` in `level-blockout.md` as an engine range. Source inspection shows `$rangePattern` has no engine-context requirement.
