---
id: B-package-fab-actor-board-reference
title: "Public actor guide fails the package text gate"
status: OPEN
severity: Medium
category: bug
tags: [packaging, docs, actor]
encounters: 1
lastSeen: 2026-08-13T00:00:00Z
---

# Public actor guide fails the package text gate

`Scripts/package-fab.ps1 -ValidateOnly` stages `Docs/wiki-src/actor.md` and
then rejects it because the public guide contains the blocked internal phrase
`board ticket`. As a result, the supported prebuilt packaging workflow stops
before `BuildPlugin` even when plugin source compiles cleanly.

**Fix:** Keep the useful example but describe it as a reproduced filter trap
without exposing the internal issue-board reference.

## History
- `#1-package-gate-repro` `OPEN` reporter — Reproduced through `package-prebuilt.ps1`: staging stopped in `Assert-StagedTextIsPublic` on `docs/wiki-src/actor.md` before compilation. A direct UE 5.8 `BuildPlugin` of the same source succeeded all 1043 actions.
