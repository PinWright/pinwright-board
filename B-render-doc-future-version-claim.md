---
id: B-render-doc-future-version-claim
title: "Render guide claims an unpublished future version"
status: IN-REVIEW
severity: Low
category: bug
tags: [docs, packaging, render, version]
encounters: 1
lastSeen: 2026-08-13T00:00:00Z
---

# Render guide claims an unpublished future version

The current plugin descriptor and generated product facts publish version
`0.7.0`, while the public render guide twice says the current orthographic-width
behavior began in `v0.7.1`. The package facts gate correctly rejects this as a
future product-version claim.

**Fix:** Preserve the useful compatibility warning but describe the previous
semantics as "earlier builds" instead of attaching an unpublished version.

## History
- `#1-future-version-package-failure` `OPEN` reporter — Reproduced after prior package gates passed: `package-fab.ps1 -ValidateOnly -KeepStage` rejects `v0.7.1` in `render.md` because `PinWright.uplugin` and `product-facts.json` both say `0.7.0`.
- `#2-use-unversioned-history-note` `IN-REVIEW` developer — Replaced both unpublished `v0.7.1` labels with "earlier builds" while preserving the raw-zoom compatibility warning. Full `package-fab.ps1 -ValidateOnly -KeepStage` now passes: 906 public files copied, 633 excluded, plugin-reference coverage validated. Published as PinWright commit `fc1a6a16`.
