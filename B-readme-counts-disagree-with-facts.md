---
id: B-readme-counts-disagree-with-facts
title: "README quoted three operation totals: a stale hand-typed overview figure and an unlabelled public-only list total, neither matching product-facts.json"
status: IN-REVIEW
severity: Low
category: bug
tags: [scripts, product-facts, readme, release, published-facts]
encounters: 1
lastSeen: 2026-09-30T00:00:00Z
---

# README totals disagreed with product-facts.json

After regenerating from a registry reporting 1,270 operations / 68 namespaces, `README.md` still
carried two other figures:

- The overview sentence ("1,263 operations across 68 namespaces, developed against 5,323 automated
  tests") was hand-typed at the 0.8.0 release and never touched by `scripts/gen-product-facts.ps1`,
  so it went stale on every registry change.
- The generated namespace list read "All 67 namespaces (1269 operations)" because the script filters
  out `tier == 'internal'` namespaces (`pipeline`, 1 operation) without saying so, so the list
  total could not be reconciled with the published 1,270 / 68.

**Impact:** a public surface quoting numbers that disagree with the fact source; no code defect.
**Fix:** the script rewrites the overview sentence from the same facts (throws if the sentence is
reworded, so it cannot go stale silently), labels the list "public namespaces", and names every
omitted internal namespace with its operation count.

## History
- `#1-three-readme-totals` `OPEN` reporter — Found while regenerating facts: README overview said 1,263 / 68 / 5,323, list said 67 / 1,269, `product-facts.json` says 1,270 / 68 / 5,408. The 1-namespace / 1-operation gap is the internal-tier `pipeline` namespace, filtered at `gen-product-facts.ps1` (`$listed`) without disclosure.
- `#2-script-owns-readme-totals` `IN-REVIEW` developer — Plugin commit `04d86b1c`: `gen-product-facts.ps1` regex-rewrites the overview sentence from `operations` / `namespaceCount` / `tests`, heads the list "All N public namespaces", adds "Not listed: internal plumbing namespaces ... `pipeline` (1 operation)"; `docs/release-checklist.md` step 1 updated. Verified: two consecutive Windows PowerShell 5.1 runs produce byte-identical README; README now reads 1,270 / 68 / 5,408 and 67 public (1,269) + `pipeline` (1).
