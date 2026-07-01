---
id: B-replace-sig-mismatch
title: "`compile_bpir` replace mode doesn't replace on signature mismatch"
status: DONE
severity: High
category: bug
tags: []
---

# `compile_bpir` replace mode doesn't replace on signature mismatch

`mode: "replace"` with different signature errors same as default mode. Should unconditionally delete old entry + subgraph, then create fresh.

## History
- `#1-same-error-default-mode` `OPEN` reporter — Tested with TestDupe(string) → TestDupe(int), got same error as default mode.
- `#2-verified-replace-different-sig` `DONE` tester — Verified: created TestEnum() on W_PhotoPopup, then replace-mode compiled TestEnum(int NewParam) with different signature. Old event deleted, new one created, BP compiled clean. No errors.
