---
id: F-decompile-enum-names
title: "Decompiler outputs raw integers for enum pin defaults"
status: DONE
severity: ""
category: feature
tags: []
---

# Decompiler outputs raw integers for enum pin defaults

Pin has correct `pinSubType` but decompiler emits `B: 2` instead of `B: EDroneArmState::ArmedWaitingEngines`. Breaks round-tripping — decompiled BPIR fed back into compile_bpir reproduces B-enum-raw-integers.

**Fix:** Check `pinSubType` for UEnum in decompiler, call `GetNameStringByValue()`.

## History
- `#1-raw-integer-enum-output` `OPEN` reporter — Decompiled OnDroneArmedEvent showed B:0 for both comparisons instead of enum names.
- `#2-verified-enum-names` `DONE` tester — Verified: decompiled W_PhotoPopup shows ESlateVisibility::Collapsed, ESlateVisibility::SelfHitTestInvisible, ESlateVisibility::Hidden as full enum names. Round-trip confirmed.
