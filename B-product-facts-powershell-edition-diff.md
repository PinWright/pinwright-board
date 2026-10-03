---
id: B-product-facts-powershell-edition-diff
title: "gen-product-facts.ps1 formats product-facts.json with ConvertTo-Json, whose layout differs between PowerShell 7 and Windows PowerShell 5.1, so a PS7 run re-indents the whole file"
status: OPEN
severity: Low
category: bug
tags: [scripts, product-facts, powershell, release, determinism, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
rice: [1, 1, 1, 1]
priority: 8
---

# product-facts.json output depends on the PowerShell edition

`scripts/gen-product-facts.ps1:272` writes the facts file as
`($facts | ConvertTo-Json -Depth 5) + "`r`n"`. `ConvertTo-Json`'s layout is edition-specific: Windows
PowerShell 5.1 emits 4-space indents, two spaces after each colon and arrays aligned under their key
(the committed `product-facts.json` has exactly that shape: `"version":  "0.8.0"`, array items indented
to column 24), while PowerShell 7 emits 2-space indents and one space after the colon. Running the
script under `pwsh` therefore rewrites every line of the 371-line file (reported as a ~738-line diff:
every line removed and re-added) even when no fact changed, burying the real number changes the file
exists to review. `docs/release-checklist.md:59` documents the 5.1 invocation
(`powershell -ExecutionPolicy Bypass -File ...`), but nothing in the script enforces it.

Verified by reading the script and the committed file's format; the PS7 run itself was reported by
today's verification, not re-run here (the script overwrites the tracked file).

**Impact:** review noise on a release artifact; no wrong number. Pure friction.
**Fix:** make the output independent of the edition. Smallest options: add `#Requires -PSEdition
Desktop` (Windows-only, matching the documented invocation), or post-process the JSON into one fixed
layout before `Write-TextFile`. Either way, a second run under the other edition must produce a
zero-byte diff.

## History
- `#1-edition-dependent-json` `OPEN` reporter — Found in today's verification (PS7 run produced a whole-file re-indent). Source-verified: `gen-product-facts.ps1:272` uses `ConvertTo-Json`; committed file carries the 5.1 layout.
