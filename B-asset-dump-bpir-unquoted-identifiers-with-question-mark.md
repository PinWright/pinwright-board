---
id: B-asset-dump-bpir-unquoted-identifiers-with-question-mark
title: "asset.dump bpir.txt does not escape boolean variable names ending in '?'"
status: DONE
severity: Low
category: bug
tags: [asset-dump, bpir, escape]
---

# asset.dump bpir.txt does not escape boolean variable names ending in '?'

Boolean BP variables whose UE display names end in `?` (a UE convention for booleans like "Is Visible?") emit literal `?` in the BPIR. Round-trip parsing is fragile (regex special char) and reduces grep-ability.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/UI/W_AxisValueDisplay/bpir.txt`.
2. Observe: `select(Index: $Raw Value?, ...)`, `entry function `Set Axis Value`(double Value, bool Mode01?, ...)`.

**Fix:** Replace BPIR's space-only identifier quoting with the shared IR name-token formatter in the decompiler, and unwrap those tokens in the BPIR parser for entry params, output params, and call arg names. Do not sanitize names; preserving UE names is required for round-trip fidelity.

## History
- `#1-initial-repro` `OPEN` reporter — Boolean BP variables with display names ending `?` emit unescaped `?` in BPIR (`select(Index: $Raw Value?, ...)`, `bool Mode01?`). Sample path: `App/App/UI/W_AxisValueDisplay/bpir.txt`. Round-trip parsing fragile due to regex special-char.
- `#2-quote-bpir-name-tokens` `IN-REVIEW` developer — Reused shared IR name-token formatting for BPIR decompiler identifiers and updated BPIR parser name-token unwrapping for params/args so boolean display names ending in '?' emit backtick-quoted instead of raw. Added FBpirQuestionMarkNameTokensDecompileTest.
- `#3-verify-fix` `DONE` tester — Verified: `blueprint.decompile` on `/App/App/UI/W_AxisValueDisplay` now emits `$`Raw Value?``, `` `Mode01?`: false ``, `bool `Mode01?``, and also backtick-quotes space-only names (`` `Config Axis Type` ``, `` `Axis Index` ``, `` `Is Button` ``). Raw unquoted `?` identifiers are gone.
