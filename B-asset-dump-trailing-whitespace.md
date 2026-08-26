---
id: B-asset-dump-trailing-whitespace
title: "asset.dump emits trailing line whitespace"
status: OPEN
severity: Low
category: bug
tags: [asset-dump, text, determinism]
encounters: 1
---

# asset.dump emits trailing line whitespace

Text sidecars are written byte-for-byte from each builder, so spaces and tabs at line ends reach
the dump mirror. This creates noisy diffs and makes byte comparisons depend on builder formatting.
A writer-level fixture reproduces trailing whitespace on every supplied line.

All generated text must remove trailing spaces and tabs at the final writer boundary, including
normal dumps and diff artifacts. Binary sidecars must remain unchanged.

## History
- `#1-trailing-whitespace-reproduced` `OPEN` reporter — The production writer preserved spaces and tabs at the ends of four fixture lines; tracked output also contains affected text sidecars.
